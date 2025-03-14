# Angular Deployment and CI/CD

## Question
How do you deploy Angular applications? Explain different deployment strategies and CI/CD practices.

## Answer

### Introduction to Angular Deployment

Angular applications can be deployed using various strategies and tools. Here's a comprehensive guide to deploying Angular applications and implementing CI/CD pipelines.

### 1. Build Optimization

#### Production Build

```bash
# Generate production build
ng build --configuration production

# Analyze bundle size
ng build --stats-json
npx webpack-bundle-analyzer dist/stats.json
```

#### Build Configuration

```typescript
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "2mb",
                  "maximumError": "5mb"
                },
                {
                  "type": "anyComponentStyle",
                  "maximumWarning": "2kb",
                  "maximumError": "4kb"
                }
              ],
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.prod.ts"
                }
              ],
              "outputHashing": "all",
              "sourceMap": false,
              "namedChunks": false,
              "aot": true,
              "extractLicenses": true,
              "vendorChunk": false,
              "buildOptimizer": true,
              "optimization": true
            }
          }
        }
      }
    }
  }
}
```

### 2. Environment Configuration

#### Environment Files

```typescript
// environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000',
  featureFlags: {
    enableNewUI: true,
    enableBetaFeatures: false
  }
};

// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.production.com',
  featureFlags: {
    enableNewUI: false,
    enableBetaFeatures: false
  }
};
```

#### Environment Variables

```typescript
// environment.service.ts
@Injectable({
  providedIn: 'root'
})
export class EnvironmentService {
  private environment: any;

  constructor() {
    this.environment = environment;
  }

  getValue(key: string): any {
    return this.environment[key];
  }

  isProduction(): boolean {
    return this.environment.production;
  }
}
```

### 3. Docker Deployment

#### Dockerfile

```dockerfile
# Build stage
FROM node:16 as build

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build -- --configuration production

# Production stage
FROM nginx:alpine

COPY --from=build /app/dist/my-app /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### Nginx Configuration

```nginx
# nginx.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 1y;
        add_header Cache-Control "public, no-transform";
    }

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

### 4. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy Angular App

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '16'

    - name: Install dependencies
      run: npm ci

    - name: Run tests
      run: npm test

    - name: Build
      run: npm run build -- --configuration production

    - name: Deploy to Azure
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'my-angular-app'
        package: './dist/my-app'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
```

### 5. Deployment Strategies

#### Blue-Green Deployment

```typescript
// deployment.service.ts
@Injectable({
  providedIn: 'root'
})
export class DeploymentService {
  private currentVersion = 'blue';
  private newVersion = 'green';

  switchVersion() {
    // Switch to new version
    this.currentVersion = this.newVersion;
    this.newVersion = this.currentVersion === 'blue' ? 'green' : 'blue';
  }
}
```

#### Rolling Updates

```yaml
# kubernetes-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: angular-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: angular-app
        image: my-angular-app:latest
```

### 6. Monitoring and Logging

#### Application Monitoring

```typescript
// monitoring.service.ts
@Injectable({
  providedIn: 'root'
})
export class MonitoringService {
  constructor() {
    // Initialize monitoring
    this.initializeMonitoring();
  }

  private initializeMonitoring() {
    // Set up error tracking
    window.onerror = (msg, url, lineNo, columnNo, error) => {
      this.logError({
        message: msg,
        url,
        lineNo,
        columnNo,
        error: error?.stack
      });
      return false;
    };

    // Set up performance monitoring
    if ('performance' in window) {
      window.performance.onresourcetimingbufferfull = () => {
        this.logPerformanceMetrics();
      };
    }
  }

  private logError(error: any) {
    // Send error to monitoring service
    console.error('Error:', error);
  }

  private logPerformanceMetrics() {
    // Send performance metrics to monitoring service
    const metrics = window.performance.getEntriesByType('resource');
    console.log('Performance metrics:', metrics);
  }
}
```

### 7. Build Optimization Techniques

#### Code Splitting

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  },
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.module').then(m => m.DashboardModule)
  }
];
```

#### Preloading Strategy

```typescript
// custom-preloading-strategy.ts
@Injectable({ providedIn: 'root' })
export class CustomPreloadingStrategy implements PreloadAllModules {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (route.data?.['preload'] === false) {
      return of(null);
    }
    return load();
  }
}

// Using in routing
@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: CustomPreloadingStrategy
    })
  ]
})
export class AppRoutingModule { }
```

### 8. Deployment Checklist

```typescript
// deployment-checklist.service.ts
@Injectable({
  providedIn: 'root'
})
export class DeploymentChecklistService {
  private checklist = [
    {
      name: 'Build',
      check: () => this.checkBuild(),
      required: true
    },
    {
      name: 'Tests',
      check: () => this.checkTests(),
      required: true
    },
    {
      name: 'Linting',
      check: () => this.checkLinting(),
      required: true
    },
    {
      name: 'Bundle Size',
      check: () => this.checkBundleSize(),
      required: true
    },
    {
      name: 'Environment Variables',
      check: () => this.checkEnvironmentVariables(),
      required: true
    }
  ];

  async runChecks(): Promise<boolean> {
    for (const item of this.checklist) {
      const result = await item.check();
      if (!result && item.required) {
        return false;
      }
    }
    return true;
  }

  private async checkBuild(): Promise<boolean> {
    try {
      await exec('ng build --configuration production');
      return true;
    } catch (error) {
      console.error('Build failed:', error);
      return false;
    }
  }

  // Implement other check methods...
}
```

### Conclusion

Key points to remember:

1. **Build Optimization**:
   - Use production builds
   - Implement code splitting
   - Optimize bundle size
   - Use preloading strategies

2. **Environment Management**:
   - Configure environment files
   - Use environment variables
   - Implement feature flags

3. **Deployment Strategies**:
   - Blue-Green deployment
   - Rolling updates
   - Canary releases
   - Feature flags

4. **CI/CD Pipeline**:
   - Automated testing
   - Automated builds
   - Automated deployment
   - Environment management

5. **Monitoring and Logging**:
   - Error tracking
   - Performance monitoring
   - User analytics
   - Server monitoring

Remember to:
- Optimize build size
- Implement proper testing
- Use secure deployment practices
- Monitor application health
- Implement proper logging
- Use environment-specific configurations
- Follow deployment best practices
- Implement proper backup strategies 