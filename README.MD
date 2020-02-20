# Spring Boot + Angular + MongoDB Rest API Tutorial!
Dans ce document, nous allons vous montrer comment créer une **todo-application** à part entière avec **Spring Boot** et **MongoDB** dans le back-end et **Angular** dans le front-end.

Voici une capture d'écran de la **todo-app** que nous allons créer dans ce tutoriel.

![alt text](https://www.callicoder.com/assets/images/post/large/spring-boot-mongodb-angular-rest-api-tutorial.jpg "Todo-app")

# Outils nécessaires
* **Java JDK 8**, [Installation](https://www.oracle.com/java/technologies/javase-downloads.html)
* **Maven**, [Comment l'installer ?](https://howtodoinjava.com/maven/how-to-install-maven-on-windows/)
* **MongoDB**, [Installation](https://www.mongodb.com/download-center/community)
* **npm**, [Installation](https://nodejs.org/en/download/), c'est l'installateur du Node.js, npm est inclus avec
* **Visual studio code**, [Installation](https://code.visualstudio.com/), nous utiliserons cet IDE sur le projet Angular
* **IntelliJ IDEA Community**, [Installation](https://www.jetbrains.com/idea/download/#section=windows), nous utiliserons cet IDE sur le projet Spring Boot

## Création du back-end Spring Boot

#### 1. Génération du projet
Nous utiliserons l'application Web **Spring Initializer** pour générer notre projet **Spring Boot**.
* Rendez-vous sur http://start.spring.io/.
* Entrez la valeur de l'artefact : **todoapp**.
* Ajoutez **Spring Web** et **Spring Data MongoDB** dans la section des dépendances.
* Cliquez sur **Generate** pour générer et télécharger le projet.
![alt text](https://i.ibb.co/68hjxYw/Spring-Initializr.png "Génération du projet avec Spring initializer")
Une fois le projet téléchargé, décompressez-le et importez-le dans votre IDE préféré. La structure du répertoire du projet devrait ressembler à ceci -
![alt text](https://www.callicoder.com/assets/images/post/large/spring-boot-mongodb-angular-backend-directory-structure.jpg "Structure du projet Spring Boot")
#### 2. Configuration de la base de données MongoDB
Spring Boot essaie de configurer automatiquement la plupart des éléments pour vous en fonction des dépendances que vous avez ajoutées dans le fichier pom.xml.

Puisque nous avons ajouté la dépendance spring-boot-starter-mongodb, Spring Boot essaie d'établir une connexion avec MongoDB en lisant la configuration de la base de données à partir du fichier application.properties.

Ouvrez le fichier application.properties et ajoutez les propriétés mongodb suivantes
```properties
# MONGODB (MongoProperties)
spring.data.mongodb.uri=mongodb://localhost:27017/todoapp
```
#### 3. Création du modèle  `Todo` 
Créons maintenant le modèle `Todo` qui sera mappé à un document dans la base de données **mongodb**. Créez un nouveau package `models` dans `com.example.todoapp` et ajoutez un fichier `Todo.java` à l'intérieur du package `models` avec le contenu suivant -
```java
package com.example.todoapp.models;

import java.util.Date;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Size;
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection="todos")
@JsonIgnoreProperties(value = {"createdAt"}, allowGetters = true)
public class Todo {
    @Id
    private String id;

    @NotBlank
    @Size(max=100)
    @Indexed(unique=true)
    private String title;

    private Boolean completed = false;

    private Date createdAt = new Date();

    public Todo() {
        super();
    }

    public Todo(String title) {
        this.title = title;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public Boolean getCompleted() {
        return completed;
    }

    public void setCompleted(Boolean completed) {
        this.completed = completed;
    }

    public Date getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Date createdAt) {
        this.createdAt = createdAt;
    }

    @Override
    public String toString() {
        return String.format(
                "Todo[id=%s, title='%s', completed='%s']",
                id, title, completed);
    }
}
```
Nous avons annoté le `titre` avec l'annotation `@Indexed` et l'avons marqué comme unique. Cela crée un index unique sur le champ de titre.

De plus, nous nous assurons que le titre de la tâche n'est pas vide en l'annotant avec l'annotation `@NotBlank`.

L'annotation `@JsonIgnoreProperties` est utilisée pour ignorer le champ `createdAt` pendant la désérialisation. Nous ne voulons pas que les clients envoient la valeur `createdAt`. S'ils envoient une valeur pour ce champ, nous l'ignorerons simplement.

#### 4. Création de `TodoRepository` pour accéder à la base de données
Ensuite, nous devons créer `TodoRepository` pour accéder aux données de la base de données.

Tout d'abord, créez un nouveau package `repositories` dans `com.example.todoapp`.

Ensuite, créez une interface nommée `TodoRepository` à l'intérieur du package de référentiels avec le contenu suivant -
```java
package com.example.todoapp.repositories;

import com.example.todoapp.models.Todo;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface TodoRepository extends MongoRepository<Todo, String> {

}
```
Nous avons étendu  `TodoRepository`  avec l'interface [`MongoRepository`](http://docs.spring.io/spring-data/mongodb/docs/current/api/org/springframework/data/mongodb/repository/MongoRepository.html)  fournie par `spring-data-mongodb`. L'interface `MongoRepository`  définit des méthodes pour toutes les opérations CRUD sur le document comme  `finAll()`,  `fineOne()`,  `save()`,  `delete()`  etc.

Spring Boot branche automatiquement une implémentation de l'interface `MongoRepository`  appelé [`SimpleMongoRepository`](http://docs.spring.io/spring-data/mongodb/docs/current/api/org/springframework/data/mongodb/repository/support/SimpleMongoRepository.html) lors de l'exécution. Ainsi, toutes les méthodes CRUD définies par `MongoRepository`  sont facilement accessibles sans rien faire.

Vous pouvez consulter toutes les méthodes disponibles à partir de  [La documentation de SimpleMongoRepository](http://docs.spring.io/spring-data/mongodb/docs/current/api/org/springframework/data/mongodb/repository/support/SimpleMongoRepository.html).
#### 5. Création de l'API - `TodoController`
Enfin, créons les API qui seront exposées aux clients. Créer un nouveau package `controllers` dans `com.example.todoapp` et ajoutez un fichier `TodoController.java` dans le package `controllers` avec le code suivant -
```java
package com.example.todoapp.controllers;  
  
import javax.validation.Valid;  
import com.example.todoapp.models.Todo;  
import com.example.todoapp.repositories.TodoRepository;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.data.domain.Sort;  
import org.springframework.http.ResponseEntity;  
import org.springframework.web.bind.annotation.*;  
import java.util.List;  
  
@RestController  
@RequestMapping("/api")  
@CrossOrigin("*")  
public class TodoController {  
  
    @Autowired  
    TodoRepository todoRepository;  
  
  @GetMapping("/todos")  
    public List<Todo> getAllTodos() {  
        return todoRepository.findAll(Sort.by(Sort.Direction.DESC, "createdAt"));  
  }  
  
    @PostMapping("/todos")  
    public Todo createTodo(@Valid @RequestBody Todo todo) {  
        todo.setCompleted(false);  
 return todoRepository.save(todo);  
  }  
  
    @GetMapping(value="/todos/{id}")  
    public ResponseEntity<Todo> getTodoById(@PathVariable("id") String id) {  
        return todoRepository.findById(id)  
                .map(todo -> ResponseEntity.ok().body(todo))  
                .orElse(ResponseEntity.notFound().build());  
  }  
  
    @PutMapping(value="/todos/{id}")  
    public ResponseEntity<Todo> updateTodo(@PathVariable("id") String id,  
  @Valid @RequestBody Todo todo) {  
        return todoRepository.findById(id)  
                .map(todoData -> {  
                    todoData.setTitle(todo.getTitle());  
  todoData.setCompleted(todo.getCompleted());  
  Todo updatedTodo = todoRepository.save(todoData);  
 return ResponseEntity.ok().body(updatedTodo);  
  }).orElse(ResponseEntity.notFound().build());  
  }  
  
    @DeleteMapping(value="/todos/{id}")  
    public ResponseEntity<?> deleteTodo(@PathVariable("id") String id) {  
        return todoRepository.findById(id)  
                .map(todo -> {  
                    todoRepository.deleteById(id);  
 return ResponseEntity.ok().build();  
  }).orElse(ResponseEntity.notFound().build());  
  }  
}
```
L'annotation `@CrossOrigin` dans le contrôleur ci-dessus est utilisée pour activer les requêtes d'origine croisée. Ceci est nécessaire car nous allons accéder aux API à partir du serveur frontal angular.

D'accord! Notre travail back-end est maintenant terminé. Vous pouvez exécuter l'application en utilisant -
```bash
mvn spring-boot:run
```
L'application démarre sur le port 8080 . Vous pouvez tester les API back-end à l'aide de [Postman](https://www.postman.com/)  ou de tout autre client rest de votre choix.

Voici les tests avec l'outil [Insomnia ](https://insomnia.rest/)
* Creation d'un Todo
![alt text](https://i.ibb.co/qRKVQHG/create-Todo.png "Test createTodo")
* Lister tous les Todos
![alt text](https://i.ibb.co/tD11zTg/get-All-Todos.png "Test getTodos")
* Lister un Todo en utilisant son ID
![alt text](https://i.ibb.co/nb3fDJN/get-Todo-By-Id.png "Test getTodoById")
* Mettre a jour un Todo en utilisant son ID
![alt text](https://i.ibb.co/9cv76DN/update-Todo.png "Test updateTodo")
* Supprimer un Todo en utilisant son ID
![alt text](https://i.ibb.co/xzqCYQN/delete-Todo.png "Test deleteTodo")

Travaillons maintenant sur le front-end angular.

## Création du Front-end Angular
#### 1. Générer l'application en utilisant `angular-cli`
`Angular-cli` est un outil de ligne de commande pour créer des applications `angular` prêtes pour la production. Il aide à créer, tester, regrouper et déployer un projet `angular`.

Vous pouvez installer `angular` en tapant la commande suivante dans votre terminal -
```bash
$ npm install -g @angular/cli

```
Une fois installé, vous pouvez générer un nouveau projet en utilisant la commande `ng new` -
```bash
$ ng new todoapp-ui

```
#### 2. Exécution de l'application
Exécutons d'abord l'application générée par `angular-cli`, puis nous modifierons et ajouterons les fichiers nécessaires à notre application. Pour exécuter l'application, accédez au répertoire racine de l'application et saisissez `ng serve`.
```bash
$ cd todoapp-ui
$ ng serve --open
```
L'option `--open` est utilisée pour ouvrir automatiquement l'application dans votre navigateur par défaut. L'application se chargera dans votre navigateur par défaut à  [http://localhost:4200](http://localhost:4200) et affichera un message de bienvenue -

![alt text](https://i.ibb.co/z4SQn32/Capture.png)

#### 3. Changer le `template` pour `AppComponent` (app.component.html)

Le template de la page d'accueil par défaut que vous voyez est défini dans app.component.html. Supprimez tout dans ce fichier et ajoutez le code suivant -

```html
<main>
  <todo-list></todo-list>
</main>
```
La balise `<todo-list>` sera utilisée pour afficher le composant `TodoList` dans cette page. Nous allons définir le `TodoListComponent` bientôt.

#### 4. Créez la classe `Todo`, `todo.ts`

Avant de définir le `TodoListComponent`, définissons une classe `Todo` pour travailler avec `Todos`. créer un nouveau fichier `todo.ts` dans le dossier `src/app` et y ajouter le code suivant -
```typescript
export class Todo {
  id: string;
  title: string;
  completed: boolean;
  createdAt: Date;
}
```
#### 5. Créez `TodoListComponent`, `todo-list.component.ts`

Nous allons maintenant définir le `TodoListComponent`. Il sera utilisé pour afficher une liste de todos, créer un nouveau todo, éditer et mettre à jour un todo.
Créez un nouveau fichier `todo-list.component.ts` dans le répertoire `src/app` et ajoutez-y le code suivant -
```typescript
import { Component, Input, OnInit } from  '@angular/core';

import { TodoService } from  './todo.service';

import { Todo } from  './todo';

import {NgForm} from  '@angular/forms';

  

@Component({

selector: 'todo-list',

templateUrl: './todo-list.component.html'

})

  

export  class  TodoListComponent  implements  OnInit {

todos:  Todo[];

newTodo:  Todo  =  new  Todo();

editing:  boolean  =  false;

editingTodo:  Todo  =  new  Todo();

  

constructor(

private  todoService:  TodoService,

) {}

  

ngOnInit():  void {

this.getTodos();

}

  

getTodos():  void {

this.todoService.getTodos()

.then(todos  => this.todos = todos );

}

  

createTodo(todoForm:  NgForm):  void {

this.todoService.createTodo(this.newTodo)

.then(createTodo  => {

todoForm.reset();

this.newTodo =  new  Todo();

this.todos.unshift(createTodo)

});

}

  

deleteTodo(id:  string):  void {

this.todoService.deleteTodo(id)

.then(() => {

this.todos = this.todos.filter(todo  => todo.id != id);

});

}

  

updateTodo(todoData:  Todo):  void {

console.log(todoData);

this.todoService.updateTodo(todoData)

.then(updatedTodo  => {

let existingTodo = this.todos.find(todo  => todo.id === updatedTodo.id);

Object.assign(existingTodo, updatedTodo);

this.clearEditing();

});

}

  

toggleCompleted(todoData:  Todo):  void {

todoData.completed =  !todoData.completed;

this.todoService.updateTodo(todoData)

.then(updatedTodo  => {

let existingTodo = this.todos.find(todo  => todo.id === updatedTodo.id);

Object.assign(existingTodo, updatedTodo);

});

}

  

editTodo(todoData:  Todo):  void {

this.editing =  true;

Object.assign(this.editingTodo, todoData);

}

  

clearEditing():  void {

this.editingTodo =  new  Todo();

this.editing =  false;

}

}
```
Nous devons déclarer le `TodoListComponent` dans `app.module.ts`.
Importez simplement `TodoListComponent` et ajoutez-le au tableau de déclarations dans `@NgModule` -
```typescript
import { BrowserModule } from  '@angular/platform-browser';

import { NgModule } from  '@angular/core';

import { AppComponent } from  './app.component';

import { FormsModule } from  '@angular/forms';

import { HttpClientModule } from  '@angular/common/http';

import { TodoListComponent } from  './todo-list.component';

  

@NgModule({

declarations: [AppComponent, TodoListComponent],

imports: [BrowserModule, FormsModule, HttpClientModule],

providers: [],

bootstrap: [AppComponent]

})

export  class  AppModule {}
```
Notez que j'ai également ajouté `FormsModule` et `HttpClientModule` dans `app.module.ts`, car nous en aurons besoin pour gérer la liaison de formulaire et appeler respectivement les API REST.
#### 6. Créez un template pour `TodoListComponent`, `todo-list.component.html`
Ensuite, nous allons définir le template pour `TodoListComponent`. Créez un nouveau fichier `todo-list.component.html` dans le répertoire `src/app` et ajoutez-y le code suivant -
```html
<div class="todo-content">
    <h1 class="page-title">My Todos</h1>
    <div class="todo-create">
      <form #todoForm="ngForm" (ngSubmit)="createTodo(todoForm)" novalidate>
        <input type="text" id="title" class="form-control" placeholder="Type a todo and press enter..." 
               required
               name="title" [(ngModel)]="newTodo.title"
               #title="ngModel" >
        <div *ngIf="title.errors && title.dirty"
             class="alert alert-danger">
            <div [hidden]="!title.errors.required">
              Title is required.
            </div>
        </div>
      </form>
    </div>
    <ul class="todo-list">
      <li *ngFor="let todo of todos" [class.completed]="todo.completed === true" >
        <div class="todo-row" *ngIf="!editing || editingTodo.id != todo.id">
            <a class="todo-completed" (click)="toggleCompleted(todo)">
              <i class="material-icons toggle-completed-checkbox"></i> 
            </a> 
            <span class="todo-title">
              {{todo.title}}
            </span>
            <span class="todo-actions">
                <a (click)="editTodo(todo)">
                  <i class="material-icons edit">edit</i>
                </a>
                <a (click)="deleteTodo(todo.id)">
                  <i class="material-icons delete">clear</i>
                </a>
            </span>
        </div>
        <div class="todo-edit" *ngIf="editing && editingTodo.id === todo.id">
            <input class="form-control" type="text" 
             [(ngModel)]="editingTodo.title" required>
            <span class="edit-actions">
                <a (click)="updateTodo(editingTodo)">
                  <i class="material-icons">done</i>
                </a>
                <a (click)="clearEditing()">
                  <i class="material-icons">clear</i>
                </a>
            </span>
        </div>
      </li>
    </ul>
    <div class="no-todos" *ngIf="todos && todos.length == 0">
        <p>No Todos Found!</p>
    </div>
</div>
```
Le template contient du code pour afficher une liste de todos et des méthodes d'appel pour modifier un todo, le supprimer et le marquer comme terminé.
#### 7. Ajoutez des styles, `styles.css`
Ajoutons quelques styles pour que notre modèle soit beau. Notez que j'ajoute tous les styles dans le fichier global `styles.css`. Mais, vous pouvez définir des styles liés à un composant dans un fichier `CSS` séparé et référencer ce `CSS` en définissant `styleUrls` dans le décorateur `@Component`.
Ouvrez le fichier `styles.css` situé dans le dossier `src` et ajoutez les styles suivants -
```css
/* You can add global styles to this file, and also import other style files */

body {
  font-size: 18px;
  line-height: 1.58;
  background: #d53369;
  background: -webkit-linear-gradient(to left, #cbad6d, #d53369);
  background: linear-gradient(to left, #cbad6d, #d53369); 
  color: #333;
}

h1 {
  font-size: 36px;
}

* {
  box-sizing: border-box;
}

i {
  vertical-align: middle;
  color: #626262;
}

input {
  border: 1px solid #E8E8E8;
}

.page-title {
  text-align: center;
}

.todo-content {
  max-width: 650px;
  width: 100%;
  margin: 0 auto;
  margin-top: 60px;
  background-color: #fff;
  padding: 15px; 
  box-shadow: 0 0 4px rgba(0,0,0,.14), 0 4px 8px rgba(0,0,0,.28);
}

.form-control {
  font-size: 16px;
  padding-left: 15px;
  outline: none;
  border: 1px solid #E8E8E8;
}

.form-control:focus {
  border: 1px solid #626262;
}

.todo-content .form-control {
  width: 100%;
  height: 50px;
}

.todo-content .todo-create {
  padding-bottom: 30px;
  border-bottom: 1px solid #e8e8e8;
}

.todo-content .alert-danger {
  padding-left: 15px;
  font-size: 14px;
  color: red;
  padding-top: 5px;
}

.todo-content ul {
  list-style: none;
  margin: 0;
  padding: 0;
  max-height: 450px;
  padding-left: 15px;
  padding-right: 15px;
  margin-left: -15px;
  margin-right: -15px;
  overflow-y: scroll;
}

.todo-content ul li {
    padding-top: 10px;
    padding-bottom: 10px;
    border-bottom: 1px solid #E8E8E8;
}

.todo-content ul li span {
  display: inline-block;
  vertical-align: middle;
}

.todo-content .todo-title {
  width: calc(100% - 160px);
  margin-left: 10px;
  overflow: hidden;
  text-overflow: ellipsis;
}

.todo-content .todo-completed {
  display: inline-block;
  text-align: center;
  width:35px;
  height:35px;
  cursor: pointer;
}

.todo-content .todo-completed i {
  font-size: 20px;
}

.todo-content .todo-actions, .todo-content .edit-actions {
    float: right;
}

.todo-content .todo-actions i, .todo-content .edit-actions i {
    font-size: 17px;
}

.todo-content .todo-actions a, .todo-content .edit-actions a {
  display: inline-block;
  text-align: center;
  width: 35px;
  height: 35px;
  cursor: pointer;
}

.todo-content .todo-actions a:hover, .todo-content .edit-actions a:hover {
  background-color: #f4f4f4;
}

.todo-content .todo-edit input {
  width: calc(100% - 80px);
  height: 35px;
}
.todo-content .edit-actions {
  text-align: right;
}

.no-todos {
  text-align: center;
}

.toggle-completed-checkbox:before {
  content: 'check_box_outline_blank';
}

li.completed .toggle-completed-checkbox:before {
  content: 'check_box';
}

li.completed .todo-title {
  text-decoration: line-through;
  color: #757575;
}

li.completed i {
  color: #757575;
}
```
#### 8. Ajoutez `material-icons` dans `index.html`

Nous utilisons des icônes `material-icons` pour afficher le bouton de modification, le bouton de suppression et la case à cocher. Ajoutez simplement la balise suivante au fichier `src/index.html` -
```html
  <link rel="stylesheet" href="https://fonts.googleapis.com/icon?family=Material+Icons">

```
#### 9. Créez `TodoService`, `todo.service.ts`
Le `TodoService` sera utilisé pour obtenir les données du back-end en appelant les `API` `Spring Boot`. Créez un nouveau fichier `todo.service.ts` dans le répertoire `src/app` et ajoutez-y le code suivant -

```typescript
import { Injectable } from  '@angular/core';
import { Todo } from  './todo';
import { HttpClient } from  '@angular/common/http';

@Injectable()
export  class  TodoService {
	private baseUrl =  'http://localhost:8080';
	
	constructor(private  http:  HttpClient) { }
	
	getTodos():  Promise<Todo[]> {
		return this.http.get(this.baseUrl +  '/api/todos/')
			.toPromise()
			.then(response  => response as  Todo[])
			.catch(this.handleError);
	}

	createTodo(todoData:  Todo):  Promise<Todo> {
		return this.http.post(this.baseUrl +  '/api/todos/', todoData).
			toPromise().then(response  => response as  Todo)
			.catch(this.handleError);
	}

	updateTodo(todoData:  Todo):  Promise<Todo> {
		return this.http.put(this.baseUrl +  '/api/todos/'  + todoData.id, todoData)
			.toPromise()
			.then(response  => response as  Todo)
			.catch(this.handleError);
	}
	
	deleteTodo(id:  string):  Promise<any> {
		return this.http.delete(this.baseUrl +  '/api/todos/'  + id)
			.toPromise()
			.catch(this.handleError);
	}
	
	private  handleError(error:  any):  Promise<any> {
		console.error('Some error occured', error);
		return  Promise.reject(error.message || error);
	}
}
```
Nous devons déclarer `TodoService` dans `app.module.ts` pour pouvoir l'utiliser dans nos composants.

Ouvrez `app.module.ts` et ajoutez l'instruction d'importation suivante -
```typescript
import { TodoService } from './todo.service';
```
Ensuite, ajoutez `TodoService` dans le tableau providers-
```typescript
providers: [TodoService]
```
# Exécution de serveurs back-end et front-end et  test l'application
Vous pouvez exécuter le back-end `Spring Boot` en tapant `mvn spring-boot: run` dans le terminal. Il fonctionnera sur le port 8080.
```bash
$ cd todoapp
$ mvn spring-boot:run
```
Le front-end `angular` peut être exécuté en tapant la commande `ng serve`. Il démarrera sur le port 4200 -
```bash
$ cd angular-frontend
$ ng serve --open
```
Amusez-vous à travailler avec l'application maintenant 😊.
# Conclusion
Félicitations à vous si vous avez suivi jusqu'à la fin et construit l'application avec succès.

Si vous n'avez pas travaillé avec moi étape par étape, vous pouvez obtenir le code source de l'application entière à partir de mon repo github et y travailler.

Je sais, cet article était assez long, et je ne suis pas entré dans les détails de tous les sujets. Mais vous pouvez toujours demander de l'aide. Je serais ravi de vous aider.

Merci d'avoir lu. Happy coding!
# Sources d'apprentissage et d'inspiration
1. Documentation officielle
* [Documentation Spring Boot ](https://spring.io/quickstart)
* [Documentation ASP.NET](https://docs.microsoft.com/en-us/aspnet/)
* [Documentation Node.js](https://nodejs.org/docs/latest-v11.x/api/)
* [Documentation Laravel](https://laravel.com/docs/6.x)
* [Documentation Symfony ](https://symfony.com/doc/current/index.html#gsc.tab=0)
* [Documentation Angular ](https://angular.io/start)
* [Documentation Vue.js](https://vuejs.org/v2/guide/)
* [Documentation React](https://reactjs.org/docs/getting-started.html)
* [Documentation Android](https://developer.android.com/docs)
* [Documentation Flutter](https://flutter.dev/docs)
2. Blogs et plateformes d'apprentissage
* [Laracasts](https://laracasts.com/)
* [Callicoder](https://www.callicoder.com/)
* [Medium](https://medium.com/)
* [Edureka](https://www.edureka.co/)
* [Edx](https://www.edx.org/)
* [Coursera](https://www.coursera.org/)
* [Udemy](https://www.udemy.com/)
4. chaînes youtube
* [Traversy Media](https://www.youtube.com/channel/UC29ju8bIPH5as8OGnQzwJyA)
* [Clever Programmer](https://www.youtube.com/channel/UCqrILQNl5Ed9Dz6CGMyvMTQ)
* [Tech Wall](https://www.youtube.com/channel/UC_wmzB9ziSW5KbZxyDfZV_w)
* [Socratica](https://www.youtube.com/user/SocraticaStudios)
