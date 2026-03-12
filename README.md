# ☁️ Symfony Clean Architecture — DDD, Domain Events & Cloud-Ready

Une application Cloud-Ready n'est pas juste une application dans un container.
C'est une application dont l'architecture est conçue pour être déployée,
scalée et opérée dans un environnement cloud : services découplés, communication
asynchrone, configuration par variables d'environnement, health checks.
Ce projet est un template Symfony qui répond à ces exigences.

---

## Architecture Cloud-Ready — ce que ça signifie concrètement

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Cloud-Ready Checklist                                      │
  │                                                             │
  │  ✅ Stateless — aucun état en mémoire entre requêtes        │
  │  ✅ Config par env vars — .env.prod sans secrets hardcodés  │
  │  ✅ Docker multi-stage — image de prod légère (<100MB)      │
  │  ✅ Health check endpoint — /health pour load balancer      │
  │  ✅ Logs structurés JSON — compatibles CloudWatch/ELK       │
  │  ✅ Async via Messenger — workers découplés de l'API        │
  │  ✅ DB externalisée — RDS (AWS) en prod, SQLite en CI       │
  │  ✅ Migrations versionnées — Doctrine Migrations, idempotentes│
  └─────────────────────────────────────────────────────────────┘
```

---

## Domain Events + Symfony Messenger — communication asynchrone

```php
// User/Domain/Event/UserCreatedEvent.php
final readonly class UserCreatedEvent
{
    public function __construct(
        public string             $userId,
        public string             $email,
        public \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}
}

// User/Application/Listener/SendWelcomeEmailOnUserCreated.php
// Handler synchrone — s'exécute immédiatement dans le même processus
final readonly class SendWelcomeEmailOnUserCreated
{
    public function __construct(private MailerInterface $mailer) {}

    #[AsMessageHandler]
    public function __invoke(UserCreatedEvent $event): void
    {
        $email = (new TemplatedEmail())
            ->to($event->email)
            ->subject('Bienvenue !')
            ->htmlTemplate('emails/welcome.html.twig')
            ->context(['userId' => $event->userId]);

        $this->mailer->send($email);
    }
}

// Pour rendre l'événement ASYNCHRONE (Cloud-Ready) :
// config/packages/messenger.php
return static function (MessengerConfig $messenger) {
    $messenger->routing([
        UserCreatedEvent::class       => ['async'],  // → queue SQS (AWS) ou Redis
        IntegrationEventInterface::class => ['async'],
        SendEmailMessage::class       => ['async'],
    ]);
};

// Lancer le worker :
// php bin/console messenger:consume async --limit=100 --memory-limit=128M
```

---

## Docker multi-stage — image de production optimisée

```dockerfile
# Dockerfile
# ---- Stage 1 : builder (toutes les dépendances de dev) ----
FROM php:8.2-fpm-alpine AS builder

RUN apk add --no-cache git zip libzip-dev \
    && docker-php-ext-install pdo_mysql opcache zip

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --optimize-autoloader

COPY . .
RUN composer run-script post-install-cmd \
    && php bin/console cache:warmup --env=prod

# ---- Stage 2 : production (image légère sans composer ni dev deps) ----
FROM php:8.2-fpm-alpine AS production

RUN docker-php-ext-install pdo_mysql opcache

WORKDIR /app
COPY --from=builder /app /app

# Health check intégré
HEALTHCHECK --interval=30s --timeout=5s \
    CMD php bin/console about --env=prod || exit 1

USER www-data
EXPOSE 9000
```

---

## Configuration Cloud — variables d'environnement AWS

```yaml
# config/packages/doctrine.yaml
doctrine:
    dbal:
        url: '%env(resolve:DATABASE_URL)%'
        # En prod AWS : DATABASE_URL=mysql://user:pass@rds-endpoint:3306/hermes2
        # En CI/CD    : DATABASE_URL=sqlite:///%kernel.project_dir%/var/test.db

# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                # AWS SQS : sqs://sqs.eu-west-1.amazonaws.com/123/hermes2-queue
                # Redis   : redis://redis:6379/messages
                # Dev     : doctrine://default
```

---

## GitHub Actions — CI/CD vers AWS EC2

```yaml
# .github/workflows/deploy.yml
name: CI/CD → AWS

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.2', coverage: xdebug }

      - run: composer install
      - run: php bin/console doctrine:database:create --env=test
      - run: php bin/console doctrine:migrations:migrate --no-interaction --env=test
      - run: php bin/phpunit --coverage-clover coverage.xml
      - run: php vendor/bin/phpstan analyse src --level=8

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Build & push Docker image
        run: |
          docker build --target production -t hermes2:${{ github.sha }} .
          docker tag hermes2:${{ github.sha }} ${{ secrets.ECR_REGISTRY }}/hermes2:latest
          docker push ${{ secrets.ECR_REGISTRY }}/hermes2:latest

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1
        with:
          host:     ${{ secrets.EC2_HOST }}
          username: ec2-user
          key:      ${{ secrets.EC2_KEY }}
          script: |
            docker pull ${{ secrets.ECR_REGISTRY }}/hermes2:latest
            docker-compose -f /app/docker-compose.prod.yml up -d --force-recreate
            docker exec app php bin/console doctrine:migrations:migrate --no-interaction
```

---

## Ce que j'ai appris

Le découplage via Domain Events et Messenger change la façon dont on pense
les effets de bord. Sans Messenger, "créer un utilisateur" déclenche directement
l'envoi d'email — si le serveur SMTP est down, la création échoue. Avec Messenger
async, l'utilisateur est créé et la réponse 201 est retournée immédiatement.
L'email est envoyé par un worker séparé, qui peut réessayer en cas d'échec.

C'est exactement le type de résilience attendue dans une architecture Cloud-Ready
— les effets secondaires ne doivent jamais bloquer l'opération principale.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
