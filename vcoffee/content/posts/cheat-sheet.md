---
title: "My cheat sheet"
date: 2021-01-25T17:07:15+01:00
draft: false
featured: true
tags: [
    "code",
    "clarity",
    "angular"
]
---


Here are some steps to build a Clarity Angular App :-)
* [Step 0: Requirements installation](#step-0-install-nodejs-angular-and-golang-on-mac)
* [Step 1: Build a new app](#step-1-build-a-new-app)
* [Step 2: Add component and routing](#step-2-add-components-and-routing)
* [Step 3: Add a data model](#step-3-add-a-data-model)
* [Step 4: Add Services](#step-4-add-services)
* [Step 5: Add Clarity component to display data](#step-5-add-clarity-component-to-display-data)
* [Step 6: Create a small Web Server in Go](#step-6-create-a-small-web-server-in-go)
* [Step 7: Bundle everything in a docker image](#step-7-bundle-everything-in-a-docker-image)

# Step 0: Install Node.js, Angular and GoLang on Mac

```bash
brew install node
npm install -g @angular/cli
```

For Go, check out https://golang.org/doc/install


# Step 1: Build a new app

```bash
ng new my-app --routing=false --style=css && cd my-app && ng add @clr/angular
code . # if Visual Studio Code is installed
ng serve --live-reload --open
```

# Step 2: add components and routing

```bash
ng generate module app-routing --flat --module=app
ng generate component comp1
ng generate component comp2
```

app-component.html
```html
<div class="main-container">
  <header class="header header-6">
    <div class="branding">
      <a href="javascript:void(0)">
        <clr-icon shape="vm-bug"></clr-icon>
        <span class="title">Project Clarity</span>
      </a>
    </div>
  </header>
  <div class="content-container">
    <div class="content-area">
      <router-outlet></router-outlet>
    </div>
    <clr-vertical-nav>
      <a clrVerticalNavLink routerLink="./comp1" routerLinkActive="active">
        <clr-icon clrVerticalNavIcon shape="dashboard"></clr-icon>
        Comp1
      </a>
      <a clrVerticalNavLink routerLink="./comp2">
        <clr-icon clrVerticalNavIcon shape="vm"></clr-icon>
        Comp2
     </a>
    </clr-vertical-nav>
  </div>
</div>
```

in app-routing.module.ts:

```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Comp1Component } from './comp1/comp1.component';
import { Comp2Component } from './comp2/comp2.component';
import { RouterModule, Routes } from '@angular/router';

const routes: Routes = [
  { path: 'comp1', component: Comp1Component },
  { path: 'comp2', component: Comp2Component },
];

@NgModule({
  declarations: [],
  imports: [
    CommonModule,
    RouterModule.forRoot(routes)
  ],
  exports: [ RouterModule ]
})
export class AppRoutingModule { }

```

# Step 3: add a data model
```bash
touch models.ts
```

```typescript
export interface Cat {
    text: string;
}
```

# Step 4: add services

```bash
ng generate service cats
```

in cats.service.ts:

```typescript
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { Cat } from './models';
import { HttpClient, HttpHeaders } from '@angular/common/http';

@Injectable({
  providedIn: 'root'
})
export class CatsService {

  constructor(private http: HttpClient) { }

  getCats(): Observable<Cat[]> {
    return this.http.get<Cat[]>("https://cat-fact.herokuapp.com/facts")
  }
}
```

in app.module.ts:
```typescript

import { HttpClientModule } from '@angular/common/http';

@NgModule({
  imports: [
    HttpClientModule,
  ],
})

```

in comp1.component.ts:
```typescript
import { Component, OnInit } from '@angular/core';
import { CatsService } from '../cats.service';
import { Cat } from '../models';

@Component({
  selector: 'app-comp1',
  templateUrl: './comp1.component.html',
  styleUrls: ['./comp1.component.css']
})
export class Comp1Component implements OnInit {

  constructor(public catsService: CatsService) { }

  cats: Cat[] = [];

  ngOnInit() {
    this.catsService.getCats().subscribe((data) => this.cats = data)
  }

}

```

# Step 5: add Clarity component to display data

in comp1.component.html:
```html
<p>comp1 works!</p>

<clr-datagrid>
    <clr-dg-column>Cat Facts</clr-dg-column>
  
    <clr-dg-row *ngFor="let cat of cats">
      <clr-dg-cell>{{cat.text}}</clr-dg-cell>
    </clr-dg-row>
  
    <clr-dg-footer>{{cats.length}} cat facts</clr-dg-footer>
</clr-datagrid>
```

![My App](/my-app.png)


# Step 6: create a small Web Server in Go

```bash
mkdir -p ./server
cd server
go mod init antoine
touch main.go
```

```go
package main

import (
	"github.com/gin-gonic/gin"

	"path"
	"path/filepath"
)

func main() {
	router := gin.Default()

	router.NoRoute(func(c *gin.Context) {
		dir, file := path.Split(c.Request.RequestURI)
		ext := filepath.Ext(file)
		if file == "" || ext == "" {
			c.File("./index.html")
		} else {
			c.File("./" + path.Join(dir, file))
		}

	})

	err := router.Run(":80")
	if err != nil {
		panic(err)
	}
}
```

```bash
go build
ng build --prod
mv antoine ./dist/my-app
cd ./dist/my-app
./antoine

adeleporte@adeleporte-a01 my-app % ./antoine
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] Listening and serving HTTP on :80
[GIN] 2021/01/25 - 17:02:33 | 304 |     843.929µs |             ::1 | GET      "/"
[GIN] 2021/01/25 - 17:02:33 | 200 |    5.702677ms |             ::1 | GET      "/runtime-es2015.cdfb0ddb511f65fdc0a0.js"
[GIN] 2021/01/25 - 17:02:33 | 200 |    10.11552ms |             ::1 | GET      "/styles.f763236cada916dd7938.css"
[GIN] 2021/01/25 - 17:02:33 | 200 |    5.357246ms |             ::1 | GET      "/polyfills-es2015.ffa9bb4e015925544f91.js"
[GIN] 2021/01/25 - 17:02:33 | 200 |    8.684829ms |             ::1 | GET      "/main-es2015.c04813a259ecd0c49d48.js"
[GIN] 2021/01/25 - 17:02:33 | 200 |    9.210793ms |             ::1 | GET      "/scripts.b41015bd1035aa6847f4.js"
[GIN] 2021/01/25 - 17:02:33 | 200 |     745.624µs |             ::1 | GET      "/favicon.ico"

```

# Step 7: bundle everything in a docker image

```bash
touch Dockerfile

```

```docker
FROM golang:latest as builder

LABEL maintainer="Antoine Deleporte <adeleporte@vmware.com>"

WORKDIR /app

COPY server/go.mod server/go.sum ./

RUN go mod download

COPY server .

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

FROM alpine:latest  

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/main .

COPY dist/my-app .

EXPOSE 80

CMD ["./main"]
```


```bash
docker build  -t adeleporte/my-app .

Sending build context to Docker daemon  335.1MB
Step 1/14 : FROM golang:latest as builder
 ---> 5f46b413e8f5
Step 2/14 : LABEL maintainer="Antoine Deleporte <adeleporte@vmware.com>"
 ---> Using cache
 ---> 7a87c5261969
Step 3/14 : WORKDIR /app
 ---> Using cache
 ---> d2966e96f2a6
Step 4/14 : COPY server/go.mod server/go.sum ./
 ---> Using cache
 ---> 14b26c1beb91
Step 5/14 : RUN go mod download
 ---> Using cache
 ---> ceaf8bdb094c
Step 6/14 : COPY server .
 ---> 84f589d27e31
Step 7/14 : RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .
 ---> Running in b0bff9e3e2a6
Removing intermediate container b0bff9e3e2a6
 ---> 77d0de075c47
Step 8/14 : FROM alpine:latest
 ---> 389fef711851
Step 9/14 : RUN apk --no-cache add ca-certificates
 ---> Using cache
 ---> b76400e5614f
Step 10/14 : WORKDIR /root/
 ---> Using cache
 ---> 948f33f39270
Step 11/14 : COPY --from=builder /app/main .
 ---> 164380f6ce76
Step 12/14 : COPY dist/my-app .
 ---> 50ae351a17f9
Step 13/14 : EXPOSE 80
 ---> Running in 7f24383dae0f
Removing intermediate container 7f24383dae0f
 ---> 21c0ff0499c4
Step 14/14 : CMD ["./main"]
 ---> Running in 9e55a07fe09d
Removing intermediate container 9e55a07fe09d
 ---> ee88bd66f307
Successfully built ee88bd66f307
Successfully tagged adeleporte/my-app:latest

docker run -p 80:80 -l test adeleporte/my-app
docker push adeleporte/my-app
```


### *DISCLAIMER: The views and opinions expressed on this blog are our own and may not reflect the views and opinions of our employer*
