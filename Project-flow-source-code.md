# Complete Angular CRUD Implementation with TypeScript

Based on your requirement, here's the complete implementation of an Employee Management System in Angular with TypeScript.

---

## 1. Project Setup & Structure

```bash
# Install Angular CLI
npm install -g @angular/cli

# Create new project
ng new employee-management --routing --style=scss

cd employee-management

# Generate folder structure
ng g module core
ng g module shared
ng g module features/employee --routing

# Generate components
ng g component features/employee/employee-list
ng g component features/employee/employee-add
ng g component features/employee/employee-edit
ng g component features/employee/employee-view

# Generate services
ng g service core/services/employee
ng g service core/services/auth

# Generate models
ng g interface models/employee
ng g interface models/auth-response

# Generate guards
ng g guard core/guards/auth

# Generate interceptors
ng g interceptor core/interceptors/auth
ng g interceptor core/interceptors/error

# Generate shared components
ng g component shared/components/loading-spinner
ng g component shared/components/confirmation-dialog
```

---

## 2. Models (Interfaces)

### `src/app/models/employee.ts`

```typescript
export interface Employee {
  id?: number;
  name: string;
  email: string;
  department: string;
  salary: number;
  phone?: string;
  address?: string;
  hireDate?: Date;
  createdAt?: Date;
  updatedAt?: Date;
}

export interface EmployeeResponse {
  success: boolean;
  data: Employee | Employee[];
  message?: string;
  totalCount?: number;
}
```

### `src/app/models/auth-response.ts`

```typescript
export interface LoginRequest {
  username: string;
  password: string;
}

export interface LoginResponse {
  token: string;
  user: {
    id: number;
    name: string;
    email: string;
    role: string;
  };
}

export interface AuthState {
  isLoggedIn: boolean;
  user: any;
  token: string | null;
}
```

---

## 3. Environment Configuration

### `src/environments/environment.ts`

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  authEnabled: true,
  pagination: {
    defaultPageSize: 10,
    pageSizeOptions: [5, 10, 20, 50]
  }
};
```

### `src/environments/environment.prod.ts`

```typescript
export const environment = {
  production: true,
  apiUrl: 'https://api.yourdomain.com/api',
  authEnabled: true,
  pagination: {
    defaultPageSize: 10,
    pageSizeOptions: [5, 10, 20, 50]
  }
};
```

---

## 4. App Configuration

### `src/app/app.config.ts`

```typescript
import { ApplicationConfig, importProvidersFrom } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideAnimations } from '@angular/platform-browser/animations';
import { ReactiveFormsModule } from '@angular/forms';

import { routes } from './app.routes';
import { authInterceptor } from './core/interceptors/auth.interceptor';
import { errorInterceptor } from './core/interceptors/error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor])
    ),
    provideAnimations(),
    importProvidersFrom(ReactiveFormsModule)
  ]
};
```

---

## 5. Routing Configuration

### `src/app/app.routes.ts`

```typescript
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: '/employees',
    pathMatch: 'full'
  },
  {
    path: 'login',
    loadComponent: () => import('./features/auth/login/login.component')
      .then(m => m.LoginComponent)
  },
  {
    path: 'employees',
    loadChildren: () => import('./features/employee/employee.routes')
      .then(m => m.employeeRoutes),
    canActivate: [authGuard]
  },
  {
    path: '**',
    loadComponent: () => import('./shared/components/not-found/not-found.component')
      .then(m => m.NotFoundComponent)
  }
];
```

### `src/app/features/employee/employee.routes.ts`

```typescript
import { Routes } from '@angular/router';

export const employeeRoutes: Routes = [
  {
    path: '',
    component: () => import('./employee-list/employee-list.component')
      .then(m => m.EmployeeListComponent)
  },
  {
    path: 'add',
    component: () => import('./employee-add/employee-add.component')
      .then(m => m.EmployeeAddComponent)
  },
  {
    path: 'edit/:id',
    component: () => import('./employee-edit/employee-edit.component')
      .then(m => m.EmployeeEditComponent)
  },
  {
    path: 'view/:id',
    component: () => import('./employee-view/employee-view.component')
      .then(m => m.EmployeeViewComponent)
  }
];
```

---

## 6. Services

### `src/app/core/services/employee.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, catchError, map, throwError } from 'rxjs';
import { Employee, EmployeeResponse } from '../../models/employee';
import { environment } from '../../../environments/environment';

@Injectable({
  providedIn: 'root'
})
export class EmployeeService {
  private apiUrl = `${environment.apiUrl}/employees`;

  constructor(private http: HttpClient) {}

  // Get all employees with pagination, search, and sorting
  getEmployees(params?: {
    page?: number;
    limit?: number;
    search?: string;
    sortBy?: string;
    sortOrder?: 'asc' | 'desc';
  }): Observable<{ data: Employee[]; totalCount: number }> {
    let httpParams = new HttpParams();
    
    if (params) {
      Object.keys(params).forEach(key => {
        if (params[key as keyof typeof params] !== undefined && params[key as keyof typeof params] !== null) {
          httpParams = httpParams.set(key, String(params[key as keyof typeof params]));
        }
      });
    }

    return this.http.get<EmployeeResponse>(this.apiUrl, { params: httpParams })
      .pipe(
        map(response => ({
          data: Array.isArray(response.data) ? response.data : [],
          totalCount: response.totalCount || 0
        })),
        catchError(this.handleError)
      );
  }

  // Get single employee by ID
  getEmployee(id: number): Observable<Employee> {
    return this.http.get<EmployeeResponse>(`${this.apiUrl}/${id}`)
      .pipe(
        map(response => response.data as Employee),
        catchError(this.handleError)
      );
  }

  // Add new employee
  addEmployee(employee: Partial<Employee>): Observable<Employee> {
    return this.http.post<EmployeeResponse>(this.apiUrl, employee)
      .pipe(
        map(response => response.data as Employee),
        catchError(this.handleError)
      );
  }

  // Update employee
  updateEmployee(id: number, employee: Partial<Employee>): Observable<Employee> {
    return this.http.put<EmployeeResponse>(`${this.apiUrl}/${id}`, employee)
      .pipe(
        map(response => response.data as Employee),
        catchError(this.handleError)
      );
  }

  // Delete employee
  deleteEmployee(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`)
      .pipe(
        catchError(this.handleError)
      );
  }

  // Search employees
  searchEmployees(term: string): Observable<Employee[]> {
    return this.http.get<EmployeeResponse>(`${this.apiUrl}/search`, {
      params: { q: term }
    })
      .pipe(
        map(response => response.data as Employee[]),
        catchError(this.handleError)
      );
  }

  // Error handling
  private handleError(error: any): Observable<never> {
    let errorMessage = 'An error occurred. Please try again.';
    
    if (error.error?.message) {
      errorMessage = error.error.message;
    } else if (error.status === 0) {
      errorMessage = 'Network error. Please check your connection.';
    } else if (error.status === 404) {
      errorMessage = 'Resource not found.';
    } else if (error.status === 500) {
      errorMessage = 'Server error. Please try again later.';
    }

    return throwError(() => new Error(errorMessage));
  }
}
```

### `src/app/core/services/auth.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable, tap, catchError, throwError } from 'rxjs';
import { Router } from '@angular/router';
import { LoginRequest, LoginResponse, AuthState } from '../../models/auth-response';
import { environment } from '../../../environments/environment';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private apiUrl = `${environment.apiUrl}/auth`;
  private authState = new BehaviorSubject<AuthState>({
    isLoggedIn: false,
    user: null,
    token: null
  });

  authState$ = this.authState.asObservable();

  constructor(
    private http: HttpClient,
    private router: Router
  ) {
    this.loadTokenFromStorage();
  }

  login(credentials: LoginRequest): Observable<LoginResponse> {
    return this.http.post<LoginResponse>(`${this.apiUrl}/login`, credentials)
      .pipe(
        tap(response => {
          this.setSession(response);
        }),
        catchError(this.handleError)
      );
  }

  logout(): void {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
    this.authState.next({
      isLoggedIn: false,
      user: null,
      token: null
    });
    this.router.navigate(['/login']);
  }

  isLoggedIn(): boolean {
    return this.authState.value.isLoggedIn && !!this.getToken();
  }

  getToken(): string | null {
    return localStorage.getItem('token');
  }

  getUser(): any {
    const userStr = localStorage.getItem('user');
    return userStr ? JSON.parse(userStr) : null;
  }

  private setSession(response: LoginResponse): void {
    localStorage.setItem('token', response.token);
    localStorage.setItem('user', JSON.stringify(response.user));
    
    this.authState.next({
      isLoggedIn: true,
      user: response.user,
      token: response.token
    });
  }

  private loadTokenFromStorage(): void {
    const token = this.getToken();
    const user = this.getUser();
    
    if (token && user) {
      this.authState.next({
        isLoggedIn: true,
        user,
        token
      });
    }
  }

  private handleError(error: any): Observable<never> {
    let errorMessage = 'Login failed. Please try again.';
    
    if (error.status === 401) {
      errorMessage = 'Invalid username or password.';
    } else if (error.status === 0) {
      errorMessage = 'Network error. Please check your connection.';
    }

    return throwError(() => new Error(errorMessage));
  }
}
```

---

## 7. Interceptors

### `src/app/core/interceptors/auth.interceptor.ts`

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../services/auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  if (token) {
    const cloned = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`)
    });
    return next(cloned);
  }

  return next(req);
};
```

### `src/app/core/interceptors/error.interceptor.ts`

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error) => {
      if (error.status === 401) {
        authService.logout();
        router.navigate(['/login']);
      }
      
      return throwError(() => error);
    })
  );
};
```

---

## 8. Guards

### `src/app/core/guards/auth.guard.ts`

```typescript
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard = () => {
  const router = inject(Router);
  const authService = inject(AuthService);

  if (authService.isLoggedIn()) {
    return true;
  }

  return router.parseUrl('/login');
};
```

---

## 9. Employee Components

### `src/app/features/employee/employee-list/employee-list.component.ts`

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { FormsModule } from '@angular/forms';
import { EmployeeService } from '../../../core/services/employee.service';
import { Employee } from '../../../models/employee';
import { Subscription } from 'rxjs';
import { environment } from '../../../../environments/environment';

@Component({
  selector: 'app-employee-list',
  standalone: true,
  imports: [CommonModule, RouterModule, FormsModule],
  templateUrl: './employee-list.component.html',
  styleUrls: ['./employee-list.component.scss']
})
export class EmployeeListComponent implements OnInit, OnDestroy {
  employees: Employee[] = [];
  loading = false;
  error = '';
  searchTerm = '';

  // Pagination
  currentPage = 1;
  pageSize = environment.pagination.defaultPageSize;
  totalItems = 0;
  totalPages = 0;

  // Sorting
  sortBy = 'id';
  sortOrder: 'asc' | 'desc' = 'asc';

  private subscriptions = new Subscription();

  constructor(private employeeService: EmployeeService) {}

  ngOnInit(): void {
    this.loadEmployees();
  }

  loadEmployees(): void {
    this.loading = true;
    this.error = '';

    const params = {
      page: this.currentPage,
      limit: this.pageSize,
      search: this.searchTerm || undefined,
      sortBy: this.sortBy,
      sortOrder: this.sortOrder
    };

    this.subscriptions.add(
      this.employeeService.getEmployees(params).subscribe({
        next: (response) => {
          this.employees = response.data;
          this.totalItems = response.totalCount;
          this.totalPages = Math.ceil(this.totalItems / this.pageSize);
          this.loading = false;
        },
        error: (error) => {
          this.error = error.message || 'Failed to load employees';
          this.loading = false;
        }
      })
    );
  }

  deleteEmployee(id: number, name: string): void {
    if (confirm(`Are you sure you want to delete ${name}?`)) {
      this.loading = true;
      this.subscriptions.add(
        this.employeeService.deleteEmployee(id).subscribe({
          next: () => {
            this.loadEmployees();
            // Show success toast (you can add a toast service)
          },
          error: (error) => {
            this.error = error.message || 'Failed to delete employee';
            this.loading = false;
          }
        })
      );
    }
  }

  onSearch(): void {
    this.currentPage = 1;
    this.loadEmployees();
  }

  clearSearch(): void {
    this.searchTerm = '';
    this.currentPage = 1;
    this.loadEmployees();
  }

  onPageChange(page: number): void {
    this.currentPage = page;
    this.loadEmployees();
  }

  onSort(column: string): void {
    if (this.sortBy === column) {
      this.sortOrder = this.sortOrder === 'asc' ? 'desc' : 'asc';
    } else {
      this.sortBy = column;
      this.sortOrder = 'asc';
    }
    this.loadEmployees();
  }

  getSortIcon(column: string): string {
    if (this.sortBy !== column) return '↕';
    return this.sortOrder === 'asc' ? '↑' : '↓';
  }

  getPageNumbers(): number[] {
    const pages: number[] = [];
    const maxVisible = 5;
    let start = Math.max(1, this.currentPage - Math.floor(maxVisible / 2));
    let end = Math.min(this.totalPages, start + maxVisible - 1);
    
    if (end - start < maxVisible - 1) {
      start = Math.max(1, end - maxVisible + 1);
    }

    for (let i = start; i <= end; i++) {
      pages.push(i);
    }
    
    return pages;
  }

  ngOnDestroy(): void {
    this.subscriptions.unsubscribe();
  }
}
```

### `src/app/features/employee/employee-list/employee-list.component.html`

```html
<div class="employee-list-container">
  <div class="header">
    <h2>Employee Management</h2>
    <button class="btn btn-primary" routerLink="/employees/add">
      + Add Employee
    </button>
  </div>

  <!-- Search Bar -->
  <div class="search-container">
    <div class="search-box">
      <input
        type="text"
        [(ngModel)]="searchTerm"
        placeholder="Search employees..."
        (keyup.enter)="onSearch()"
      />
      <button class="btn btn-search" (click)="onSearch()">Search</button>
      <button *ngIf="searchTerm" class="btn btn-clear" (click)="clearSearch()">✕</button>
    </div>
  </div>

  <!-- Error Message -->
  <div *ngIf="error" class="alert alert-danger">
    {{ error }}
  </div>

  <!-- Loading Spinner -->
  <div *ngIf="loading" class="loading-overlay">
    <div class="spinner"></div>
  </div>

  <!-- Employee Table -->
  <div class="table-responsive" *ngIf="!loading">
    <table class="employee-table">
      <thead>
        <tr>
          <th (click)="onSort('id')">
            ID {{ getSortIcon('id') }}
          </th>
          <th (click)="onSort('name')">
            Name {{ getSortIcon('name') }}
          </th>
          <th (click)="onSort('email')">
            Email {{ getSortIcon('email') }}
          </th>
          <th (click)="onSort('department')">
            Department {{ getSortIcon('department') }}
          </th>
          <th (click)="onSort('salary')">
            Salary {{ getSortIcon('salary') }}
          </th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngIf="employees.length === 0">
          <td colspan="6" class="no-data">No employees found</td>
        </tr>
        <tr *ngFor="let employee of employees">
          <td>{{ employee.id }}</td>
          <td>{{ employee.name }}</td>
          <td>{{ employee.email }}</td>
          <td>{{ employee.department }}</td>
          <td>{{ employee.salary | currency }}</td>
          <td class="actions">
            <button class="btn btn-view" [routerLink]="['/employees/view', employee.id]">
              View
            </button>
            <button class="btn btn-edit" [routerLink]="['/employees/edit', employee.id]">
              Edit
            </button>
            <button class="btn btn-delete" (click)="deleteEmployee(employee.id!, employee.name)">
              Delete
            </button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>

  <!-- Pagination -->
  <div class="pagination-container" *ngIf="totalItems > 0">
    <div class="pagination-info">
      Showing {{ (currentPage - 1) * pageSize + 1 }} - 
      {{ Math.min(currentPage * pageSize, totalItems) }} of {{ totalItems }}
    </div>
    <div class="pagination-controls">
      <button 
        class="btn" 
        [disabled]="currentPage === 1"
        (click)="onPageChange(currentPage - 1)"
      >
        Previous
      </button>
      
      <button
        *ngFor="let page of getPageNumbers()"
        class="btn"
        [class.active]="page === currentPage"
        (click)="onPageChange(page)"
      >
        {{ page }}
      </button>
      
      <button 
        class="btn" 
        [disabled]="currentPage === totalPages"
        (click)="onPageChange(currentPage + 1)"
      >
        Next
      </button>
    </div>
  </div>
</div>
```

### `src/app/features/employee/employee-list/employee-list.component.scss`

```scss
.employee-list-container {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;

  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;

    h2 {
      margin: 0;
      color: #333;
    }
  }

  .search-container {
    margin-bottom: 20px;

    .search-box {
      display: flex;
      gap: 10px;
      max-width: 500px;

      input {
        flex: 1;
        padding: 10px;
        border: 1px solid #ddd;
        border-radius: 4px;
        font-size: 14px;

        &:focus {
          outline: none;
          border-color: #007bff;
        }
      }
    }
  }

  .employee-table {
    width: 100%;
    border-collapse: collapse;
    background: white;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);

    th {
      background: #f8f9fa;
      padding: 12px 16px;
      text-align: left;
      font-weight: 600;
      color: #333;
      cursor: pointer;
      user-select: none;

      &:hover {
        background: #e9ecef;
      }
    }

    td {
      padding: 12px 16px;
      border-bottom: 1px solid #f1f1f1;
    }

    tr:hover td {
      background: #f8f9fa;
    }

    .no-data {
      text-align: center;
      padding: 40px;
      color: #999;
    }

    .actions {
      display: flex;
      gap: 8px;

      .btn {
        padding: 4px 12px;
        font-size: 12px;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        transition: all 0.2s;

        &.btn-view {
          background: #17a2b8;
          color: white;
        }

        &.btn-edit {
          background: #ffc107;
          color: #333;
        }

        &.btn-delete {
          background: #dc3545;
          color: white;
        }

        &:hover {
          opacity: 0.8;
          transform: translateY(-1px);
        }
      }
    }
  }

  .pagination-container {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 20px;
    padding: 16px 0;

    .pagination-info {
      color: #666;
      font-size: 14px;
    }

    .pagination-controls {
      display: flex;
      gap: 5px;

      .btn {
        padding: 6px 12px;
        border: 1px solid #ddd;
        background: white;
        cursor: pointer;
        border-radius: 4px;
        transition: all 0.2s;

        &:hover:not(:disabled) {
          background: #007bff;
          color: white;
          border-color: #007bff;
        }

        &.active {
          background: #007bff;
          color: white;
          border-color: #007bff;
        }

        &:disabled {
          opacity: 0.5;
          cursor: not-allowed;
        }
      }
    }
  }

  .loading-overlay {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 200px;

    .spinner {
      width: 40px;
      height: 40px;
      border: 3px solid #f3f3f3;
      border-top: 3px solid #007bff;
      border-radius: 50%;
      animation: spin 1s linear infinite;
    }
  }

  .alert {
    padding: 12px 16px;
    border-radius: 4px;
    margin-bottom: 16px;

    &.alert-danger {
      background: #f8d7da;
      color: #721c24;
      border: 1px solid #f5c6cb;
    }
  }

  .btn-primary {
    background: #007bff;
    color: white;
    padding: 10px 20px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;

    &:hover {
      background: #0056b3;
    }
  }

  .btn-search {
    background: #28a745;
    color: white;
    padding: 10px 20px;
    border: none;
    border-radius: 4px;
    cursor: pointer;

    &:hover {
      background: #218838;
    }
  }

  .btn-clear {
    background: #dc3545;
    color: white;
    padding: 10px 12px;
    border: none;
    border-radius: 4px;
    cursor: pointer;

    &:hover {
      background: #c82333;
    }
  }
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
```

---

## 10. Add Employee Component

### `src/app/features/employee/employee-add/employee-add.component.ts`

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Router, RouterModule } from '@angular/router';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { EmployeeService } from '../../../core/services/employee.service';
import { Employee } from '../../../models/employee';

@Component({
  selector: 'app-employee-add',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
  templateUrl: './employee-add.component.html',
  styleUrls: ['./employee-add.component.scss']
})
export class EmployeeAddComponent implements OnInit {
  employeeForm!: FormGroup;
  loading = false;
  submitted = false;
  error = '';

  constructor(
    private formBuilder: FormBuilder,
    private employeeService: EmployeeService,
    private router: Router
  ) {}

  ngOnInit(): void {
    this.employeeForm = this.formBuilder.group({
      name: ['', [Validators.required, Validators.minLength(2), Validators.maxLength(50)]],
      email: ['', [Validators.required, Validators.email]],
      department: ['', [Validators.required]],
      salary: ['', [Validators.required, Validators.min(0)]],
      phone: ['', [Validators.pattern('^[0-9]{10}$')]],
      address: ['', [Validators.maxLength(200)]],
      hireDate: [new Date().toISOString().split('T')[0], [Validators.required]]
    });
  }

  get f() {
    return this.employeeForm.controls;
  }

  onSubmit(): void {
    this.submitted = true;
    this.error = '';

    if (this.employeeForm.invalid) {
      return;
    }

    this.loading = true;
    const employeeData: Partial<Employee> = {
      ...this.employeeForm.value,
      salary: Number(this.employeeForm.value.salary)
    };

    this.employeeService.addEmployee(employeeData).subscribe({
      next: () => {
        this.loading = false;
        // Navigate to employee list
        this.router.navigate(['/employees']);
        // You can add a success toast here
      },
      error: (error) => {
        this.error = error.message || 'Failed to add employee';
        this.loading = false;
      }
    });
  }

  resetForm(): void {
    this.employeeForm.reset({
      hireDate: new Date().toISOString().split('T')[0]
    });
    this.submitted = false;
    this.error = '';
  }
}
```

### `src/app/features/employee/employee-add/employee-add.component.html`

```html
<div class="employee-form-container">
  <div class="form-header">
    <h2>Add New Employee</h2>
    <button class="btn btn-secondary" routerLink="/employees">
      Back to List
    </button>
  </div>

  <div *ngIf="error" class="alert alert-danger">
    {{ error }}
  </div>

  <form [formGroup]="employeeForm" (ngSubmit)="onSubmit()">
    <div class="form-grid">
      <div class="form-group">
        <label for="name">Full Name *</label>
        <input 
          type="text" 
          id="name"
          formControlName="name"
          class="form-control"
          [class.is-invalid]="submitted && f['name'].invalid"
        />
        <div *ngIf="submitted && f['name'].invalid" class="invalid-feedback">
          <div *ngIf="f['name'].errors?.['required']">Name is required</div>
          <div *ngIf="f['name'].errors?.['minlength']">Name must be at least 2 characters</div>
          <div *ngIf="f['name'].errors?.['maxlength']">Name cannot exceed 50 characters</div>
        </div>
      </div>

      <div class="form-group">
        <label for="email">Email *</label>
        <input 
          type="email" 
          id="email"
          formControlName="email"
          class="form-control"
          [class.is-invalid]="submitted && f['email'].invalid"
        />
        <div *ngIf="submitted && f['email'].invalid" class="invalid-feedback">
          <div *ngIf="f['email'].errors?.['required']">Email is required</div>
          <div *ngIf="f['email'].errors?.['email']">Please enter a valid email</div>
        </div>
      </div>

      <div class="form-group">
        <label for="department">Department *</label>
        <select 
          id="department"
          formControlName="department"
          class="form-control"
          [class.is-invalid]="submitted && f['department'].invalid"
        >
          <option value="">Select Department</option>
          <option value="Engineering">Engineering</option>
          <option value="HR">Human Resources</option>
          <option value="Finance">Finance</option>
          <option value="Marketing">Marketing</option>
          <option value="Sales">Sales</option>
          <option value="IT">IT</option>
        </select>
        <div *ngIf="submitted && f['department'].invalid" class="invalid-feedback">
          <div *ngIf="f['department'].errors?.['required']">Department is required</div>
        </div>
      </div>

      <div class="form-group">
        <label for="salary">Salary *</label>
        <input 
          type="number" 
          id="salary"
          formControlName="salary"
          class="form-control"
          [class.is-invalid]="submitted && f['salary'].invalid"
        />
        <div *ngIf="submitted && f['salary'].invalid" class="invalid-feedback">
          <div *ngIf="f['salary'].errors?.['required']">Salary is required</div>
          <div *ngIf="f['salary'].errors?.['min']">Salary must be greater than 0</div>
        </div>
      </div>

      <div class="form-group">
        <label for="phone">Phone</label>
        <input 
          type="tel" 
          id="phone"
          formControlName="phone"
          class="form-control"
          [class.is-invalid]="submitted && f['phone'].invalid"
          placeholder="Enter 10-digit phone number"
        />
        <div *ngIf="submitted && f['phone'].invalid" class="invalid-feedback">
          <div *ngIf="f['phone'].errors?.['pattern']">Please enter a valid 10-digit phone number</div>
        </div>
      </div>

      <div class="form-group">
        <label for="hireDate">Hire Date *</label>
        <input 
          type="date" 
          id="hireDate"
          formControlName="hireDate"
          class="form-control"
          [class.is-invalid]="submitted && f['hireDate'].invalid"
        />
        <div *ngIf="submitted && f['hireDate'].invalid" class="invalid-feedback">
          <div *ngIf="f['hireDate'].errors?.['required']">Hire date is required</div>
        </div>
      </div>

      <div class="form-group full-width">
        <label for="address">Address</label>
        <textarea 
          id="address"
          formControlName="address"
          class="form-control"
          rows="3"
          [class.is-invalid]="submitted && f['address'].invalid"
        ></textarea>
        <div *ngIf="submitted && f['address'].invalid" class="invalid-feedback">
          <div *ngIf="f['address'].errors?.['maxlength']">Address cannot exceed 200 characters</div>
        </div>
      </div>
    </div>

    <div class="form-actions">
      <button 
        type="submit" 
        class="btn btn-primary"
        [disabled]="loading"
      >
        {{ loading ? 'Saving...' : 'Save Employee' }}
      </button>
      <button 
        type="button" 
        class="btn btn-secondary"
        (click)="resetForm()"
      >
        Reset
      </button>
    </div>
  </form>
</div>
```

---

## 11. Edit Employee Component

### `src/app/features/employee/employee-edit/employee-edit.component.ts`

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Router, ActivatedRoute, RouterModule } from '@angular/router';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { EmployeeService } from '../../../core/services/employee.service';
import { Employee } from '../../../models/employee';

@Component({
  selector: 'app-employee-edit',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
  templateUrl: './employee-edit.component.html',
  styleUrls: ['./employee-edit.component.scss']
})
export class EmployeeEditComponent implements OnInit {
  employeeForm!: FormGroup;
  loading = false;
  submitted = false;
  error = '';
  employeeId!: number;
  employee: Employee | null = null;

  constructor(
    private formBuilder: FormBuilder,
    private employeeService: EmployeeService,
    private router: Router,
    private route: ActivatedRoute
  ) {}

  ngOnInit(): void {
    this.employeeId = Number(this.route.snapshot.paramMap.get('id'));
    
    if (!this.employeeId) {
      this.router.navigate(['/employees']);
      return;
    }

    this.initializeForm();
    this.loadEmployeeData();
  }

  initializeForm(): void {
    this.employeeForm = this.formBuilder.group({
      name: ['', [Validators.required, Validators.minLength(2), Validators.maxLength(50)]],
      email: ['', [Validators.required, Validators.email]],
      department: ['', [Validators.required]],
      salary: ['', [Validators.required, Validators.min(0)]],
      phone: ['', [Validators.pattern('^[0-9]{10}$')]],
      address: ['', [Validators.maxLength(200)]],
      hireDate: ['', [Validators.required]]
    });
  }

  loadEmployeeData(): void {
    this.loading = true;
    this.employeeService.getEmployee(this.employeeId).subscribe({
      next: (employee) => {
        this.employee = employee;
        // Patch form values
        this.employeeForm.patchValue({
          name: employee.name,
          email: employee.email,
          department: employee.department,
          salary: employee.salary,
          phone: employee.phone || '',
          address: employee.address || '',
          hireDate: employee.hireDate 
            ? new Date(employee.hireDate).toISOString().split('T')[0] 
            : ''
        });
        this.loading = false;
      },
      error: (error) => {
        this.error = error.message || 'Failed to load employee data';
        this.loading = false;
      }
    });
  }

  get f() {
    return this.employeeForm.controls;
  }

  onSubmit(): void {
    this.submitted = true;
    this.error = '';

    if (this.employeeForm.invalid) {
      return;
    }

    this.loading = true;
    const employeeData: Partial<Employee> = {
      ...this.employeeForm.value,
      salary: Number(this.employeeForm.value.salary)
    };

    this.employeeService.updateEmployee(this.employeeId, employeeData).subscribe({
      next: () => {
        this.loading = false;
        this.router.navigate(['/employees']);
      },
      error: (error) => {
        this.error = error.message || 'Failed to update employee';
        this.loading = false;
      }
    });
  }
}
```

---

## 12. View Employee Component

### `src/app/features/employee/employee-view/employee-view.component.ts`

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Router, ActivatedRoute, RouterModule } from '@angular/router';
import { EmployeeService } from '../../../core/services/employee.service';
import { Employee } from '../../../models/employee';

@Component({
  selector: 'app-employee-view',
  standalone: true,
  imports: [CommonModule, RouterModule],
  templateUrl: './employee-view.component.html',
  styleUrls: ['./employee-view.component.scss']
})
export class EmployeeViewComponent implements OnInit {
  employee: Employee | null = null;
  loading = false;
  error = '';
  employeeId!: number;

  constructor(
    private employeeService: EmployeeService,
    private router: Router,
    private route: ActivatedRoute
  ) {}

  ngOnInit(): void {
    this.employeeId = Number(this.route.snapshot.paramMap.get('id'));
    
    if (!this.employeeId) {
      this.router.navigate(['/employees']);
      return;
    }

    this.loadEmployeeData();
  }

  loadEmployeeData(): void {
    this.loading = true;
    this.employeeService.getEmployee(this.employeeId).subscribe({
      next: (employee) => {
        this.employee = employee;
        this.loading = false;
      },
      error: (error) => {
        this.error = error.message || 'Failed to load employee data';
        this.loading = false;
      }
    });
  }
}
```

### `src/app/features/employee/employee-view/employee-view.component.html`

```html
<div class="employee-view-container">
  <div class="view-header">
    <h2>Employee Details</h2>
    <div class="header-actions">
      <button class="btn btn-secondary" routerLink="/employees">Back</button>
      <button class="btn btn-edit" [routerLink]="['/employees/edit', employeeId]">Edit</button>
    </div>
  </div>

  <div *ngIf="loading" class="loading-overlay">
    <div class="spinner"></div>
  </div>

  <div *ngIf="error" class="alert alert-danger">
    {{ error }}
  </div>

  <div *ngIf="employee && !loading" class="employee-details">
    <div class="detail-card">
      <div class="detail-row">
        <label>ID:</label>
        <span>{{ employee.id }}</span>
      </div>
      <div class="detail-row">
        <label>Name:</label>
        <span>{{ employee.name }}</span>
      </div>
      <div class="detail-row">
        <label>Email:</label>
        <span>{{ employee.email }}</span>
      </div>
      <div class="detail-row">
        <label>Department:</label>
        <span>{{ employee.department }}</span>
      </div>
      <div class="detail-row">
        <label>Salary:</label>
        <span>{{ employee.salary | currency }}</span>
      </div>
      <div class="detail-row" *ngIf="employee.phone">
        <label>Phone:</label>
        <span>{{ employee.phone }}</span>
      </div>
      <div class="detail-row" *ngIf="employee.address">
        <label>Address:</label>
        <span>{{ employee.address }}</span>
      </div>
      <div class="detail-row">
        <label>Hire Date:</label>
        <span>{{ employee.hireDate | date:'mediumDate' }}</span>
      </div>
      <div class="detail-row" *ngIf="employee.createdAt">
        <label>Created At:</label>
        <span>{{ employee.createdAt | date:'medium' }}</span>
      </div>
    </div>
  </div>
</div>
```

---

## 13. Login Component

### `src/app/features/auth/login/login.component.ts`

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Router, RouterModule } from '@angular/router';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { AuthService } from '../../../core/services/auth.service';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.scss']
})
export class LoginComponent {
  loginForm: FormGroup;
  loading = false;
  submitted = false;
  error = '';

  constructor(
    private formBuilder: FormBuilder,
    private authService: AuthService,
    private router: Router
  ) {
    // Redirect if already logged in
    if (this.authService.isLoggedIn()) {
      this.router.navigate(['/employees']);
    }

    this.loginForm = this.formBuilder.group({
      username: ['', [Validators.required]],
      password: ['', [Validators.required, Validators.minLength(6)]]
    });
  }

  get f() {
    return this.loginForm.controls;
  }

  onSubmit(): void {
    this.submitted = true;
    this.error = '';

    if (this.loginForm.invalid) {
      return;
    }

    this.loading = true;
    this.authService.login(this.loginForm.value).subscribe({
      next: () => {
        this.loading = false;
        this.router.navigate(['/employees']);
      },
      error: (error) => {
        this.error = error.message || 'Login failed. Please try again.';
        this.loading = false;
      }
    });
  }
}
```

### `src/app/features/auth/login/login.component.html`

```html
<div class="login-container">
  <div class="login-card">
    <h2>Employee Management</h2>
    <h3>Login</h3>

    <div *ngIf="error" class="alert alert-danger">
      {{ error }}
    </div>

    <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
      <div class="form-group">
        <label for="username">Username</label>
        <input 
          type="text" 
          id="username"
          formControlName="username"
          class="form-control"
          [class.is-invalid]="submitted && f['username'].invalid"
          placeholder="Enter your username"
        />
        <div *ngIf="submitted && f['username'].invalid" class="invalid-feedback">
          <div *ngIf="f['username'].errors?.['required']">Username is required</div>
        </div>
      </div>

      <div class="form-group">
        <label for="password">Password</label>
        <input 
          type="password" 
          id="password"
          formControlName="password"
          class="form-control"
          [class.is-invalid]="submitted && f['password'].invalid"
          placeholder="Enter your password"
        />
        <div *ngIf="submitted && f['password'].invalid" class="invalid-feedback">
          <div *ngIf="f['password'].errors?.['required']">Password is required</div>
          <div *ngIf="f['password'].errors?.['minlength']">Password must be at least 6 characters</div>
        </div>
      </div>

      <button 
        type="submit" 
        class="btn btn-primary btn-block"
        [disabled]="loading"
      >
        {{ loading ? 'Logging in...' : 'Login' }}
      </button>
    </form>
  </div>
</div>
```

---

## 14. App Component

### `src/app/app.component.html`

```html
<nav class="navbar" *ngIf="authService.isLoggedIn()">
  <div class="nav-container">
    <div class="nav-brand">
      <a routerLink="/employees">Employee Management</a>
    </div>
    <div class="nav-menu">
      <a routerLink="/employees" routerLinkActive="active">Employees</a>
      <button class="btn btn-logout" (click)="logout()">Logout</button>
    </div>
  </div>
</nav>

<main>
  <router-outlet></router-outlet>
</main>
```

### `src/app/app.component.ts`

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule, Router } from '@angular/router';
import { AuthService } from './core/services/auth.service';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule, RouterModule],
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.scss']
})
export class AppComponent {
  title = 'employee-management';

  constructor(
    public authService: AuthService,
    private router: Router
  ) {}

  logout(): void {
    this.authService.logout();
  }
}
```

---

## 15. Main Styles

### `src/styles.scss`

```scss
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background: #f5f7fa;
  color: #333;
}

// Navbar
.navbar {
  background: #2c3e50;
  color: white;
  padding: 0 20px;
  height: 64px;
  display: flex;
  align-items: center;

  .nav-container {
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .nav-brand {
    a {
      color: white;
      text-decoration: none;
      font-size: 20px;
      font-weight: 600;
    }
  }

  .nav-menu {
    display: flex;
    align-items: center;
    gap: 20px;

    a {
      color: white;
      text-decoration: none;
      padding: 8px 16px;
      border-radius: 4px;
      transition: background 0.2s;

      &:hover {
        background: rgba(255,255,255,0.1);
      }

      &.active {
        background: rgba(255,255,255,0.2);
      }
    }
  }
}

main {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}

// Buttons
.btn {
  display: inline-block;
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
  text-decoration: none;

  &:hover {
    transform: translateY(-1px);
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  }

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
    transform: none;
  }

  &-primary {
    background: #007bff;
    color: white;

    &:hover:not(:disabled) {
      background: #0056b3;
    }
  }

  &-secondary {
    background: #6c757d;
    color: white;

    &:hover:not(:disabled) {
      background: #5a6268;
    }
  }

  &-danger {
    background: #dc3545;
    color: white;

    &:hover:not(:disabled) {
      background: #c82333;
    }
  }

  &-success {
    background: #28a745;
    color: white;

    &:hover:not(:disabled) {
      background: #218838;
    }
  }

  &-warning {
    background: #ffc107;
    color: #333;

    &:hover:not(:disabled) {
      background: #e0a800;
    }
  }

  &-info {
    background: #17a2b8;
    color: white;

    &:hover:not(:disabled) {
      background: #138496;
    }
  }

  &-logout {
    background: #dc3545;
    color: white;
    padding: 6px 16px;

    &:hover {
      background: #c82333;
    }
  }

  &-block {
    width: 100%;
    padding: 12px;
    font-size: 16px;
  }
}

// Forms
.form-group {
  margin-bottom: 16px;

  label {
    display: block;
    margin-bottom: 4px;
    font-weight: 500;
  }

  .form-control {
    width: 100%;
    padding: 10px;
    border: 1px solid #ced4da;
    border-radius: 4px;
    font-size: 14px;
    transition: border-color 0.2s;

    &:focus {
      outline: none;
      border-color: #007bff;
      box-shadow: 0 0 0 0.2rem rgba(0,123,255,0.25);
    }

    &.is-invalid {
      border-color: #dc3545;
    }
  }

  textarea.form-control {
    resize: vertical;
    min-height: 80px;
  }
}

.invalid-feedback {
  color: #dc3545;
  font-size: 12px;
  margin-top: 4px;
}

.alert {
  padding: 12px 16px;
  border-radius: 4px;
  margin-bottom: 16px;

  &-danger {
    background: #f8d7da;
    color: #721c24;
    border: 1px solid #f5c6cb;
  }

  &-success {
    background: #d4edda;
    color: #155724;
    border: 1px solid #c3e6cb;
  }

  &-info {
    background: #d1ecf1;
    color: #0c5460;
    border: 1px solid #bee5eb;
  }
}

// Loading spinner
.spinner {
  display: inline-block;
  width: 40px;
  height: 40px;
  border: 3px solid #f3f3f3;
  border-top: 3px solid #007bff;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-overlay {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 200px;
}

// Form Layout
.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;

  .full-width {
    grid-column: 1 / -1;
  }
}

.form-actions {
  margin-top: 24px;
  display: flex;
  gap: 12px;
}
```

---

## 16. Main Entry Point

### `src/main.ts`

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig)
  .catch((err) => console.error(err));
```

---

## 17. Testing

### `src/app/features/employee/employee-list/employee-list.component.spec.ts`

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { HttpClientTestingModule } from '@angular/common/http/testing';
import { RouterTestingModule } from '@angular/router/testing';
import { EmployeeListComponent } from './employee-list.component';
import { EmployeeService } from '../../../core/services/employee.service';
import { of } from 'rxjs';

describe('EmployeeListComponent', () => {
  let component: EmployeeListComponent;
  let fixture: ComponentFixture<EmployeeListComponent>;
  let employeeService: EmployeeService;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        HttpClientTestingModule,
        RouterTestingModule,
        EmployeeListComponent
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(EmployeeListComponent);
    component = fixture.componentInstance;
    employeeService = TestBed.inject(EmployeeService);
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should load employees on init', () => {
    const mockEmployees = {
      data: [
        { id: 1, name: 'John Doe', email: 'john@example.com', department: 'Engineering', salary: 50000 }
      ],
      totalCount: 1
    };

    spyOn(employeeService, 'getEmployees').and.returnValue(of(mockEmployees));
    component.ngOnInit();
    expect(employeeService.getEmployees).toHaveBeenCalled();
    expect(component.employees.length).toBe(1);
  });
});
```

---

## 18. Package Dependencies

### `package.json`

```json
{
  "name": "employee-management",
  "version": "1.0.0",
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "watch": "ng build --watch --configuration development",
    "test": "ng test"
  },
  "private": true,
  "dependencies": {
    "@angular/animations": "^17.0.0",
    "@angular/common": "^17.0.0",
    "@angular/compiler": "^17.0.0",
    "@angular/core": "^17.0.0",
    "@angular/forms": "^17.0.0",
    "@angular/platform-browser": "^17.0.0",
    "@angular/platform-browser-dynamic": "^17.0.0",
    "@angular/router": "^17.0.0",
    "rxjs": "~7.8.0",
    "tslib": "^2.3.0",
    "zone.js": "~0.14.0"
  },
  "devDependencies": {
    "@angular-devkit/build-angular": "^17.0.0",
    "@angular/cli": "^17.0.0",
    "@angular/compiler-cli": "^17.0.0",
    "@types/jasmine": "~5.1.0",
    "jasmine-core": "~5.1.0",
    "karma": "~6.4.0",
    "karma-chrome-launcher": "~3.2.0",
    "karma-coverage": "~2.2.0",
    "karma-jasmine": "~5.1.0",
    "karma-jasmine-html-reporter": "~2.1.0",
    "typescript": "~5.2.2"
  }
}
```

---

## Summary

This complete Angular CRUD application includes:

1. **Proper Folder Structure** - Core, Shared, Features separation
2. **Standalone Components** - Modern Angular approach
3. **Reactive Forms** - With validation
4. **HTTP Interceptors** - Authentication and error handling
5. **Route Guards** - Protected routes
6. **CRUD Operations** - Create, Read, Update, Delete
7. **Search & Filter** - Client-side search
8. **Pagination** - With page navigation
9. **Sorting** - Table column sorting
10. **Loading States** - Spinner indicators
11. **Error Handling** - User-friendly error messages
12. **Authentication** - Login/Logout with JWT
13. **TypeScript** - Strong typing throughout
14. **SCSS** - Styled components

This follows enterprise-grade Angular development practices and is ready for production deployment.