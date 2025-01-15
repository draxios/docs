## **end-to-end guide** for creating a **brand-new Angular project** with a single component—`AdoProjectFormComponent`—that posts JSON data to **AWS API Gateway** endpoint.

1. **Project creation**  
2. **Environment configuration**  
3. **Service implementation** (`AwsApiService`)  
4. **Component code** (`AdoProjectFormComponent`)  
5. **Module setup** and **Routing**  
6. **Testing** locally  

By following these steps, you’ll have a working Angular app that can create a JSON payload and send it to AWS.

---

## 1. Prerequisites

- **Node** (LTS version, e.g., 18+ or 20+)  
- **Angular CLI** (installed globally via `npm install -g @angular/cli`)  
- **AWS API Gateway** endpoint (for example, `https://abcdef123.execute-api.us-east-1.amazonaws.com/prod`)  
- **Basic knowledge** of how to create and navigate an Angular workspace  

---

## 2. Create a New Angular Project

1. **Open a terminal** and run:
   ```bash
   ng new simple-ado-forms --routing --style=scss
   ```
   - This creates a new Angular app named `simple-ado-forms` with routing enabled and SCSS styling.
2. **Navigate** into the project folder:
   ```bash
   cd simple-ado-forms
   ```
3. **Serve** the default app:
   ```bash
   ng serve -o
   ```
   You should see the default Angular welcome page in your browser at `http://localhost:4200`.

---

## 3. Add Environment Configuration

We’ll store our AWS API Gateway URL in the environment file. This makes it easy to switch endpoints for different environments (dev, prod, etc.).

1. Open or create `src/environments/environment.ts` and add the following:

   ```ts
   // src/environments/environment.ts
   export const environment = {
     production: false,
     // Replace with your real AWS API Gateway base URL
     apiGatewayUrl: 'https://your-api-gateway.execute-api.us-east-1.amazonaws.com/prod'
   };
   ```
2. (Optional) If you have a production file `environment.prod.ts`, you can set a different URL for production there.

---

## 4. Create the AWS API Service

We’ll create an Angular service that handles HTTP requests to AWS API Gateway.

1. In the terminal, generate a service:
   ```bash
   ng generate service services/aws-api
   ```
   This creates two files: 
   - `src/app/services/aws-api.service.ts`
   - `src/app/services/aws-api.service.spec.ts` (for testing)

2. Open `aws-api.service.ts` and **replace** its contents with:

   ```ts
   // src/app/services/aws-api.service.ts
   import { Injectable } from '@angular/core';
   import { HttpClient } from '@angular/common/http';
   import { environment } from 'src/environments/environment';
   import { Observable } from 'rxjs';

   @Injectable({
     providedIn: 'root'
   })
   export class AwsApiService {
     private apiUrl = environment.apiGatewayUrl;

     constructor(private http: HttpClient) {}

     // Example method to send a POST request
     createAdoProject(payload: any): Observable<any> {
       // Adjust the endpoint path as needed for your API
       const endpoint = `${this.apiUrl}/create-ado-project`;
       return this.http.post(endpoint, payload);
     }
   }
   ```
3. This service uses `HttpClient` and environment variables to build the request URL. Make sure to include the correct path after your base URL (in this example: `/create-ado-project`).

---

## 5. Create the ADO Project Form Component

1. **Generate** the component:
   ```bash
   ng generate component components/ado-project-form
   ```
   This creates a folder `src/app/components/ado-project-form` with `.ts`, `.html`, and `.scss` files.

2. Open `ado-project-form.component.ts` and replace its contents with:

   ```ts
   // src/app/components/ado-project-form/ado-project-form.component.ts

   import { Component } from '@angular/core';
   import { AwsApiService } from 'src/app/services/aws-api.service';

   @Component({
     selector: 'app-ado-project-form',
     templateUrl: './ado-project-form.component.html',
     styleUrls: ['./ado-project-form.component.scss']
   })
   export class AdoProjectFormComponent {
     projectForm: any = {
       project_name: '',
       project_description: '',
       project_business_contact: '',
       project_support_contact: '',
       account_group: '',
       enable_boards: false,
       is_deployable: true,
       project_type: 'standard',
       repos: []
     };

     submittedJson: any;
     isSubmitting = false;
     responseMessage = '';
     errorMessage = '';

     constructor(private awsApi: AwsApiService) {}

     onSubmit(formValue: any): void {
       // Build a sample repos array
       const payload = {
         ...formValue,
         repos: [
           {
             name: 'treasury-management-admin-tool',
             code_type: 'dotnet',
             deploy: 'lambda',
             hello_world: false,
             pipelines: [],
             has_variable_groups: false
           }
         ]
       };

       this.submittedJson = payload;
       this.isSubmitting = true;
       this.responseMessage = '';
       this.errorMessage = '';

       // Call the AWS service
       this.awsApi.createAdoProject(payload).subscribe({
         next: (response: any) => {
           this.isSubmitting = false;
           this.responseMessage = 'Successfully sent data to AWS!';
           console.log('API response:', response);
         },
         error: (err: any) => {
           this.isSubmitting = false;
           this.errorMessage = 'Failed to send data to AWS.';
           console.error('Error:', err);
         }
       });
     }
   }
   ```

3. Open `ado-project-form.component.html` and replace its contents with:

   ```html
   <!-- src/app/components/ado-project-form/ado-project-form.component.html -->

   <div class="ado-project-form-container">
     <h2>ADO Project Form</h2>
     <form #adoForm="ngForm" (ngSubmit)="onSubmit(adoForm.value)">
       
       <!-- Project Name -->
       <div>
         <label for="project_name">Project Name:</label>
         <input
           id="project_name"
           name="project_name"
           [(ngModel)]="projectForm.project_name"
           required
         />
       </div>

       <!-- Project Description -->
       <div>
         <label for="project_description">Project Description:</label>
         <input
           id="project_description"
           name="project_description"
           [(ngModel)]="projectForm.project_description"
           required
         />
       </div>

       <!-- Business Contact -->
       <div>
         <label for="project_business_contact">Business Contact:</label>
         <input
           id="project_business_contact"
           name="project_business_contact"
           [(ngModel)]="projectForm.project_business_contact"
         />
       </div>

       <!-- Support Contact -->
       <div>
         <label for="project_support_contact">Support Contact:</label>
         <input
           id="project_support_contact"
           name="project_support_contact"
           [(ngModel)]="projectForm.project_support_contact"
         />
       </div>

       <!-- Account Group -->
       <div>
         <label for="account_group">Account Group:</label>
         <input
           id="account_group"
           name="account_group"
           [(ngModel)]="projectForm.account_group"
         />
       </div>

       <!-- Enable Boards -->
       <div>
         <label for="enable_boards">Enable Boards:</label>
         <input
           type="checkbox"
           id="enable_boards"
           name="enable_boards"
           [(ngModel)]="projectForm.enable_boards"
         />
       </div>

       <!-- Is Deployable -->
       <div>
         <label for="is_deployable">Is Deployable:</label>
         <input
           type="checkbox"
           id="is_deployable"
           name="is_deployable"
           [(ngModel)]="projectForm.is_deployable"
         />
       </div>

       <!-- Project Type -->
       <div>
         <label for="project_type">Project Type:</label>
         <select
           id="project_type"
           name="project_type"
           [(ngModel)]="projectForm.project_type"
         >
           <option value="standard">standard</option>
           <option value="special">special</option>
         </select>
       </div>

       <!-- Submit Button -->
       <button type="submit" [disabled]="isSubmitting">Submit</button>
     </form>

     <!-- In-flight status -->
     <div *ngIf="isSubmitting">
       <p>Sending data to AWS, please wait...</p>
     </div>

     <!-- Success message -->
     <div *ngIf="responseMessage" style="color: green;">
       <p>{{ responseMessage }}</p>
     </div>

     <!-- Error message -->
     <div *ngIf="errorMessage" style="color: red;">
       <p>{{ errorMessage }}</p>
     </div>
     
     <!-- JSON payload preview -->
     <div *ngIf="submittedJson">
       <h3>Payload Preview</h3>
       <pre>{{ submittedJson | json }}</pre>
     </div>
   </div>
   ```

4. Optionally, open `ado-project-form.component.scss` if you want to add any custom styling:

   ```scss
   /* src/app/components/ado-project-form/ado-project-form.component.scss */

   .ado-project-form-container {
     max-width: 600px;
     margin: 0 auto;

     form > div {
       margin-bottom: 10px;
     }
   }
   ```

---

## 6. Update `app-routing.module.ts` to Use the Component

Since we created this project with `--routing`, we have `app-routing.module.ts`. Let’s add a route:

```ts
// src/app/app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { AdoProjectFormComponent } from './components/ado-project-form/ado-project-form.component';

const routes: Routes = [
  { path: 'ado-form', component: AdoProjectFormComponent },
  { path: '', redirectTo: 'ado-form', pathMatch: 'full' }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

---

## 7. Update `app.module.ts`

Make sure we import `HttpClientModule` and `FormsModule`. Open `src/app/app.module.ts`:

```ts
// src/app/app.module.ts
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClientModule } from '@angular/common/http';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { AdoProjectFormComponent } from './components/ado-project-form/ado-project-form.component';

@NgModule({
  declarations: [
    AppComponent,
    AdoProjectFormComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    HttpClientModule,
    AppRoutingModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

---

## 8. Test Locally

1. **Install dependencies** (if you haven’t yet):
   ```bash
   npm install
   ```
2. **Serve** the project:
   ```bash
   ng serve -o
   ```
3. In the browser, you’ll be redirected automatically to `http://localhost:4200/`.  
   - If not, manually navigate to `http://localhost:4200/ado-form` (depending on how you set up your route).  
4. Fill out the form fields and click **Submit**:
   - You should see the JSON payload preview at the bottom.  
   - The console (in the browser’s dev tools) will show the API call.  
   - If your API Gateway is set up and reachable, you’ll see the success or error message from your AWS Lambda (or whichever backend you have).

---

## Summary

By following these steps, you now have a **fully functional** Angular application that:

1. **Collects form data** for an ADO project (project name, description, etc.).  
2. **Assembles a JSON payload** including a sample `repos` array.  
3. **Sends** that payload to an **AWS API Gateway** endpoint via a simple POST request (`AwsApiService`).  
4. **Displays** success or error messages in the UI.  

Feel free to expand on this foundation by:

- **Adding more fields** or dynamic arrays to your forms.  
- **Incorporating real logic** into your AWS backend (Lambda or other services).  
- **Enhancing** styling, validations, or authentication.  

Once it’s working locally, you can proceed with **containerizing** the app and deploying it to AWS ECS, or set up an **Azure DevOps** pipeline to build and push the Docker image to **Amazon ECR**.  
