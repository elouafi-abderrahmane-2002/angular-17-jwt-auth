# 🔐 Angular JWT — Refresh Token, Role Guards & Tests

L'authentification JWT côté Angular, c'est plus qu'un `localStorage.setItem`.
Ce projet implémente le flux complet : access token de courte durée,
refresh token automatique, gestion des rôles par route et par template,
et tests unitaires sur chaque pièce du dispositif.

---

## Flux d'authentification complet

```
  Utilisateur → Login Form
          │
          ▼
  POST /auth/login
          │
          ▼
  Serveur retourne :
  {
    accessToken:  "eyJ..." (expire dans 15 min)
    refreshToken: "eyJ..." (expire dans 7 jours)
  }
          │
          ├── accessToken  → mémoire (BehaviorSubject)
          └── refreshToken → HttpOnly Cookie (idéal) ou localStorage

  Pour chaque requête :
  ──────────────────────────────────────────────────────
  Requête → JwtInterceptor → ajoute "Authorization: Bearer {accessToken}"
          │
          ├── [200] → réponse normale
          └── [401] → token expiré
                  │
                  ├── Appel POST /auth/refresh avec refreshToken
                  │       │
                  │       ├── [200] → nouveau accessToken → rejouer la requête originale
                  │       └── [401] → logout() + redirect /login
                  │
                  └── Requêtes en attente : mises en file, rejouées après refresh
```

---

## AuthService — gestion complète des tokens

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUserSubject = new BehaviorSubject<User | null>(null);
  public currentUser$ = this.currentUserSubject.asObservable();

  constructor(private http: HttpClient, private router: Router) {
    // Restaurer l'utilisateur depuis le token au démarrage
    const token = this.getAccessToken();
    if (token && !this.isTokenExpired(token)) {
      this.currentUserSubject.next(this.decodeToken(token));
    }
  }

  login(credentials: LoginDto): Observable<AuthTokens> {
    return this.http.post<AuthTokens>('/api/auth/login', credentials).pipe(
      tap(tokens => this.handleTokens(tokens))
    );
  }

  refreshToken(): Observable<AuthTokens> {
    const refreshToken = localStorage.getItem('refresh_token');
    return this.http.post<AuthTokens>('/api/auth/refresh', { refreshToken }).pipe(
      tap(tokens => this.handleTokens(tokens))
    );
  }

  logout(): void {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    this.currentUserSubject.next(null);
    this.router.navigate(['/auth/login']);
  }

  getUserRoles(): string[] {
    const token = this.getAccessToken();
    if (!token) return [];
    const decoded = this.decodeToken(token);
    return decoded?.roles ?? [];
  }

  getAccessToken(): string | null {
    return localStorage.getItem('access_token');
  }

  isTokenExpired(token: string): boolean {
    const decoded = this.decodeToken(token);
    if (!decoded?.exp) return true;
    return Date.now() >= decoded.exp * 1000;
  }

  private handleTokens(tokens: AuthTokens): void {
    localStorage.setItem('access_token',  tokens.accessToken);
    localStorage.setItem('refresh_token', tokens.refreshToken);
    this.currentUserSubject.next(this.decodeToken(tokens.accessToken));
  }

  private decodeToken(token: string): any {
    try {
      return JSON.parse(atob(token.split('.')[1]));
    } catch {
      return null;
    }
  }
}
```

---

## Directive *hasRole — contrôle par template

```typescript
// shared/directives/has-role.directive.ts
@Directive({ selector: '[hasRole]' })
export class HasRoleDirective implements OnInit {
  @Input('hasRole') requiredRole!: string | string[];

  constructor(
    private viewContainer: ViewContainerRef,
    private templateRef: TemplateRef<any>,
    private authService: AuthService
  ) {}

  ngOnInit(): void {
    const userRoles = this.authService.getUserRoles();
    const required = Array.isArray(this.requiredRole)
      ? this.requiredRole
      : [this.requiredRole];

    if (required.some(r => userRoles.includes(r))) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
}

// Usage dans le template :
// <button *hasRole="'ADMIN'">Supprimer</button>
// <div *hasRole="['ADMIN', 'MANAGER']">Section réservée</div>
```

---

## Tests unitaires Jasmine — AuthService

```typescript
describe('AuthService', () => {
  let service: AuthService;
  let httpMock: HttpTestingController;
  let routerSpy: jasmine.SpyObj<Router>;

  beforeEach(() => {
    routerSpy = jasmine.createSpyObj('Router', ['navigate']);

    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        AuthService,
        { provide: Router, useValue: routerSpy }
      ]
    });

    service = TestBed.inject(AuthService);
    httpMock = TestBed.inject(HttpTestingController);
    localStorage.clear();
  });

  afterEach(() => {
    httpMock.verify();
    localStorage.clear();
  });

  it('should store tokens after login', () => {
    const mockTokens = {
      accessToken:  'eyJ.mock.access',
      refreshToken: 'eyJ.mock.refresh'
    };

    service.login({ username: 'admin', password: 'secret' }).subscribe();

    const req = httpMock.expectOne('/api/auth/login');
    req.flush(mockTokens);

    expect(localStorage.getItem('access_token')).toBe(mockTokens.accessToken);
    expect(localStorage.getItem('refresh_token')).toBe(mockTokens.refreshToken);
  });

  it('should clear tokens and redirect on logout', () => {
    localStorage.setItem('access_token',  'some.token');
    localStorage.setItem('refresh_token', 'some.refresh');

    service.logout();

    expect(localStorage.getItem('access_token')).toBeNull();
    expect(routerSpy.navigate).toHaveBeenCalledWith(['/auth/login']);
  });

  it('should return user roles from decoded token', () => {
    // Token avec payload { roles: ['ADMIN', 'USER'] }
    const payload = btoa(JSON.stringify({ roles: ['ADMIN', 'USER'], exp: 9999999999 }));
    localStorage.setItem('access_token', `eyJ.${payload}.sig`);

    const roles = service.getUserRoles();

    expect(roles).toContain('ADMIN');
    expect(roles).toContain('USER');
  });
});
```

---

## Ce que j'ai appris

Le test le plus important ici n'est pas le login — c'est le comportement
sous charge simultanée : que se passe-t-il si 3 requêtes reçoivent un 401
au même moment ? Sans protection, on déclenche 3 refresh, le 2ème invalide
le token du 1er, et l'utilisateur se retrouve déconnecté sans raison.

Le `BehaviorSubject` dans l'interceptor + le flag `isRefreshing` est la
solution classique. Mais la valider en test nécessite de simuler des appels
concurrents avec `forkJoin` — c'est ce type de test qui révèle les vraies
fragilités d'une implémentation.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
