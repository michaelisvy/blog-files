This post is a guest post by [Han Lim](https://twitter.com/tagore79) and [Tony Nguyen](https://twitter.com/sgdevblog). Han and Tony have done a great presentation at our Singapore Spring User Group on Spring + Angular JS. This blog is based on their presentation.


### Target Audience

This article is written for Spring Web MVC developers who are familiar with JSP development and who would like to understand how to migrate to a client side Javascript framework like AngularJS.

### Sample Petclinic for reference

We have migrated part of the Spring Petclinic application to AngularJS (with a new design). Our branch can be found [here](https://github.com/singularity-sg/spring-petclinic). 

### Introduction to AngularJS

AngularJS is a Javascript framework created at Google that touts itself as a "Superheroic Web MVW Framework" (where the "W" in the "MVW" being a tongue-in-cheek reference to "Whatever" for all the various [MVx architectures](http://blogs.k10world.com/technology/difference-between-mvc-vs-mvp-vs-mvvm/). As it is based on an MVx architecture, AngularJS provides a structure to Javascript development and thus gives Javascript an elevated status compared to traditional Spring + JSP applications that only use Javascript to provide that bit of interactivity on the user interface. 

With AngularJS, your Javascript-based view layer also inherits features like Dependency-Injection, HTML-vocabulary extension (via the use of custom directives), unit-testing and functional testing integration as well as DOM-selectors ala JQuery (using [jqlite](https://thinkster.io/a-better-way-to-learn-angularjs/jqlite-angular-element-and-the-dom) as it provides only a subset of JQuery but you could also easily use JQuery if you prefer). AngularJS also introduces scopes to your Javascript code so that variables declared in your code are bound only to the scope that is required.This prevents variables pollution that inadvertently arises when the size of your Javascript grows. 

When you are developing a Spring Web MVC application using JSP, you will likely use the Spring-provided form tags to bind your form inputs to a server side model. Similarly, AngularJS provides a way to bind form inputs to models on the client side. In fact, it provides instantaneous 2-way data-binding from the form input to your model on the Javascript application. That means that not only do you have the benefits of having your view updated with changes inside your Javascript model, any changes you make to your UI will also update the Javascript model (and consequently any other views that is bound to that model). It is almost magical to see all the views that are bound to the same JS model on the app update the model automatically. 

Moreover, since your model can be set to a particular scope, only views that belong to the same scope will be affected, allowing you to sandbox code that should be local only to a particular portion of your view. (This is done via an AngularJS attribute called `ng-controller` that is set in your HTML templates). You can see the difference in a later section comparing JSP tags and AngularJS directives.

### Two-way Data Binding

In a Spring-JSP web application, there is one way data binding from Spring model to jsp view. Any change to the model will be reflected to Jsp view but not the reverse. This is the nature of web applications. If we build a desktop application, it is possible to do reverse data binding with Swing UI. 

However, for a web application exposing REST resources, there may be no direct data binding. Data is sent from the server to the browser as JSON objects. Without AngularJS and the like, developers need to write javascript code in order to bind javascript object to html controls. 

Because manual data binding is a tedious task, some developers try to automate the task by creating a Javascript framework for data binding. It is worth remembering that this data binding happens on the client side and the model for data binding is a Javascript object rather than a server side model. 

Angular pushes this idea further by creating a two-way binding. Changing values in an  HTML control will be reflected in the object in real time. 

![Scope](https://github.com/michaelisvy/blog-images/raw/master/01-han-tony-angularjs/scope.png)

Binding is a useful concept if you need to deal with complex UI components like AJAX tables. 

For example: we need to render a list of users and roles in an AngularJs application, with the following html template:

```html
<tr ng-repeat="user in users">
	<td>{{user.username}}</td>
	<td>{{user.role}}</td>
</tr>
...
<a ng-click="addUser()">Add new user</a>
```

The code to add a user can be this simple:

```javascript
$scope.addUser = function(){
	newUser = {}
	$scope.users.push(newUser );
}
```

If the array `users` has one more element, the table will automatically have one more row.

### Template

Using AngularJS, it is possible to write relatively complex User Interfaces in an organized and elegant manner, always encapsulating the required logic within your components and never running the risk of errant global Javascript variables polluting your scope. It is also very testable, and there are built-in mechanisms to perform tests at the unit and functional level, ensuring that your User Interface codebase goes through the same rigorous testing that your Java/Spring code undergoes, ensuring quality even at the user interface level.  

Another advantage of using AngularJS to write your html templates is that the templates are essentially similar to html even with the various front end logic baked into your view. It is possible to incorporate AngularJS logic into your template and still do a client-side validation control. In the JSP world, you can try viewing a JSP file from a browser with all the template logic in place and most likely your browser will give up rendering the page. 
You can see how a typical AngularJS template looks like :

```html
<div class="row thumbnail-wrapper">
  <div data-ng-repeat="pet in currentOwner.pets" class="col-md-3">
    <div class="thumbnail">
      <img data-ng-src="images/pets/pet{{pet.id % 10 + 1}}.jpg" 
        class="img-circle" alt="My Pet Image">
      <div class="caption">
        <h3 class="caption-heading" data-ng-bind="pet.name"></h3>
        <p class="caption-meta" data-ng-bind="pet.birthdate"></p>
        <p class="caption-meta"><span class="caption-label" 
           data-ng-bind="pet.type.name"></span></p>
      </div>
      <div class="action-bar">
        <a class="btn btn-default" data-toggle="modal" data-target="#petModal" 
          data-ng-click="editPet(pet.id)">
          <span class="glyphicon glyphicon-edit"></span> Edit Pet
        </a>
        <a class="btn btn-default">
          <span></span> Add Visit
        </a>
      </div>
    </div>
  </div>
</div>
```
You can probably spot some non-HTML additions to the template. It includes attributes like `data-ng-click` which maps a click on a button to a method name call. There's also `data-ng-repeat` which loops through a JSON array and generates the necessary html code to render the same view for each item in the array. Yet with all the logic in place, we are still able to validate and view the html template from the browser. 
AngularJS calls all the non-html tags and attributes "directives" and the purpose of these directives is to enhance the capabilities of HTML. AngularJS also supports both HTML 4 and 5 so if you have templates that are still relying on HTML 4 DOCTYPEs, it should still work fine (although the validators for HTML 4 will not recognize data-ng-x attributes).

One big difference between using AngularJS and JSP is the **rendering time**. If you use JSPs, the server renders html content. In contrast, if you use AngularJS, the rendering is happening in browser. Therefore, both the templates and JSON objects are to be sent to client side. It is worth to notice  that AngularJS may briefly display the template before running DOM manipulation to generate content. For example, if AngularJS has not completed loaded, the date of birth in the page will be shown with an empty value before showing the real value.

### Scopes in AngularJS

One important concept to grasp in AngularJS is that of scopes. In the past, whenever I had to write Javascript for my web application, I had to manage the variable names and construct special name-spaced objects in order to store my scoped properties. However, AngularJS does it for you automatically based on its MVx concept. Every directive will inherit a scope from its controller (or if you would like, an isolated scope that does not inherit other scope properties). The properties and variables created in this scope do not pollute the rest of the scopes or global context.

Scopes are used as the "glue" of an AngularJS application. Controllers in AngularJS use scopes to interact with the views. Scopes are also used to pass models and properties between directives and controllers. The advantage of this is that we are now forced to design our application in a way that components are self-contained and relationships between components have to be considered carefully through a use of a model that can be prototypically inherited from a parent scope.

A scope can be nested in another scope prototypically in the same way Javascript implements its inheritance model via prototyping. However, any property name that is declared in the child scope that is similar to the parent will hide the parent property from the child scope thereafter. An example of this can be described in the code below:

```html
<!DOCTYPE html>
<html>

  <head>
    <script data-require="angular.js@*" data-semver="1.4.0-rc.0" src="https://code.angularjs.org/1.4.0-rc.0/angular.js"></script>
    <link rel="stylesheet" href="style.css" />
    <script src="script.js"></script>
  </head>

  <body data-ng-app="demo">
    <h1>Scopes in AngularJS</h1>
    <div data-ng-controller="parentController">
      <div data-ng-controller="childController">
        <span>This is a demonstration of scopes</span>
        <div>
          Parent model: <span data-ng-bind="$parent.model.name"></span>
        </div>
        <div>
          Current model: <span data-ng-bind="model.name"></span>
        </div>
        <div>
          <button data-ng-click="updateModel()">Click me</button>
        </div>
      </div>
    </div>
  </body>

</html>
```


At the very top in the hierarchy of scopes is the $rootScope, a scope that is accessible globally and can be used as the last resort to share properties and models across the whole application. The use of this should be minimized as it introduces a sort of "global" variable that can pose the same problems when it is overused.

More information about scopes can be gleaned from the AngularJS documentation found [here](https://docs.angularjs.org/guide/scope).

### Directives in AngularJS

Directives are one of the most important concepts in AngularJS. They bring all the additional customized markup in the HTML elements, attributes, classes or comments. They are the ones giving the markup new functionalities.

The following code snippet demonstrates a customized directive called `wdsCustom` that will replace the markup element `<wds-custom company="wds">` with markup that contains information about a model called `wds`. That model element is declared in the controller scope that wraps the directive. You can have a look at the files `app.js`, `index.html` and directive template `wds-custom-directive.html` to see how this works in the plunkr snippet available [here](http://embed.plnkr.co/cP179vrMvavJieCXVe1X/preview).



As this article does not attempt to teach you how to write a directive, you can refer to the official documentation [here](https://docs.angularjs.org/guide/directive).

### Differences in architecture between JSP and AngularJS

When one migrates from a server-side templating engine like JSP or [Thymeleaf](http://www.thymeleaf.org/) to a Javascript-based templating engine, one has to experience a paradigm shift towards a client-server architecture. One has to cease thinking of the view as being a part of the web application and instead conceive the web application as 2 separate client-side and server-side applications. 
An illustration of this is in the next diagram which shows how a Spring application becomes a provider of RESTful Web Services, servicing various front end applications including an AngularJS browser-based application as well as a possibility to provide services for mobile clients like tablets or smartphones. These services could include OAuth, Authentication and other business logic services which should be obfuscated from public view. One should bear in mind that any data or business logic that is published in the form of JSON or javascript files are exposed for the client-side to see. Thus, if there's any business sensitive logic or workflow that should not be exposed, it should only be performed on the backend.


Also, when you are working in AngularJS, we prefer to encapsulate form submissions in a JSON object which is sent over to the backend RESTful service via a AngularJS HTTP Post method call (instead of using the HTML form submission). 
Validation can be performed on the front end using AngularJS's built-in or custom input validation before posting your data to the server. Of course, it is always prudent to validate the same data at the server side as well to ensure that clients that do not check their data do not compromise the integrity of the data on the server side. 


![Architecture](https://github.com/michaelisvy/blog-images/raw/master/01-han-tony-angularjs/architecture.png)



### Application structure

Let's now discuss how to organize your Spring + AngularJS application. At WDS (our company), we use Maven as our dependency and package management tool for Java/Spring and that influenced how we decided to place our AngularJS javascript application. The AngularJS application is created within `src/main/webapp` and the main files are


```
components/ # the various components are stored here.
js/app.js   # where we bootstrap the application
plugins/	# additional external plugins e.g. jquery.
services/   # common services are stored here.
images/
videos/
```



You can see an image capture of the folder structure in Eclipse below. 

![folders](https://github.com/michaelisvy/blog-images/raw/master/01-han-tony-angularjs/folder-structure.png)




Resources here are organized per the `feature-grouping` method. There are also ways to group your resources based on types, e.g. grouping all your controllers, services and views into its namesake folder. There are pros and cons for each of those options.

### Comparisons betwen AngularJS directives and JSP custom tags

If you have used Spring's custom form tags in your JSPs for developing your forms, you may be wondering if AngularJS provides the same kind of convenience for mapping form inputs to objects. The answer is yes! As a matter of fact, it is easy to bind any HTML element to a Javascript object. The only difference is that now the binding occurs on the client-side instead of the server-side.


```html
<form:form method="POST" commandName="user">
<table>
    <tr>
        <td>User Name :</td>
        <td><form:input path="name" /></td>
    </tr>
    <tr>
        <td>Password :</td>
        <td><form:password path="password" /></td>
    </tr>
    <tr>
        <td>Country :</td>
        <td>
            <form:select path="country">
            <form:option value="0" label="Select" />
            <form:options items="${countryList}" itemValue="countryId" itemLabel="countryName" />
            </form:select>
        </td>
    </tr>
</table>
</form:form>
```
Here is an example of the same form in AngularJS




```html
<form name="UserForm" data-ng-controller="ExampleUserController">
  <table>
    <tr>
        <td>User Name :</td>
        <td><input data-ng-model="user.name" /></td>
    </tr>
    <tr>
        <td>Password :</td>
        <td><input type="password" data-ng-model="user.password" /></td>
    </tr>
    <tr>
        <td>Country :</td>
        <td>
            <select data-ng-model="user.country" data-ng-options="country as country.label for country in countries">
               <option value="">Select<option />
            </select>
        </td>
    </tr>
</table>
</form>
```



Form inputs in AngularJS are augmented with additional capabilities like the `ngRequired` directive that makes the field mandatory based on certain conditions. There are also built-in validation for checking ranges, dates, patterns etc.. You can find out more at AngularJS's official documentation found [here](https://docs.angularjs.org/api/ng/input) which provides all the relevant form input directives.




### Considerations when moving from JSP to AngularJS

If you are considering migrating from JSP to AngularJS, there are a few factors to consider for a successful migration.

Converting your Spring controllers to RESTful services

You will need to transform your controllers so instead of forwarding the response to a templating engine to render a view to the client, you will provide services that will be serialized into JSON data instead. The following is an example of how a standard Spring MVC controller `RequestMapping` uses the `ModelAndView` object to render a view with the Owner as described in the url mapping.



```java
@RequestMapping("/api/owners/{ownerId}")
public ModelAndView showOwner(@PathVariable("ownerId") int ownerId) {
    ModelAndView mav = new ModelAndView("owners/ownerDetails");
    mav.addObject(this.clinicService.findOwnerById(ownerId));
    return mav;
}
```

A controller RequestMapping like that can be converted into an equivalent RESTful service that returns the owner based on the ownerId. Your template can then be moved into AngularJS which will then bind the owner object to the AngularJS template.



```java
@RequestMapping(value = "/api/owners/{id}", method = RequestMethod.GET)
public @ResponseBody Owner find(@PathVariable Integer id) {
    return this.clinicService.findOwnerById(id);
}
```

In order for Spring MVC to convert your returned object (which need to be Serializable) to a JSON object, you need to configure your Spring context file to include a conversion service that uses `Jackson2` for JSON serialization. An example of the snippet that performs this Spring context configuration is shown below.



```xml
<bean id="objectMapper" class="org.springframework.http.converter.json.Jackson2ObjectMapperFactoryBean" p:indentOutput="true" p:simpleDateFormat="yyyy-MM-dd'T'HH:mm:ss.SSSZ"></bean>
<mvc:annotation-driven conversion-service="conversionService" >
 <mvc:message-converters>
  <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter" >
   <property name="objectMapper" ref="objectMapper" />
  </bean>
 </mvc:message-converters>
</mvc:annotation-driven>
```

Once you have converted your controllers into RESTful services, you can then access these resources from your AngularJS application.

One nice way to access RESTful services in AngularJS is to use the built-in `ngResource` directive that allows you to access your RESTful services in an elegant and concise manner. An example of the Javascript code to access RESTful services using this directive can be illustrated by the following:


```javascript
var Owner = ['$resource','context', function($resource, context) {
 return $resource(context + '/api/owners/:id');
}];
 
app.factory('Owner', Owner);
 
var OwnerController = ['$scope','$state','Owner',function($scope,$state,Owner) {
 $scope.$on('$viewContentLoaded', function(event){
  $('html, body').animate({
      scrollTop: $("#owners").offset().top
  }, 1000);
 });
 
 $scope.owners = Owner.query();
}];
```

The above snippet shows how a "resource" can be created by declaring an Owner resource and then initialising it as an Owner service. The controller can then use this service to query for Owners from the RESTful endpoint. In this way, you can easily create the resources that your application require and map it easily to your business domain model. This declaration is done once only in the app.js file. You can actually take a look at this actual file in action [here](https://github.com/singularity-sg/spring-petclinic/blob/master/src/main/webapp/services/services.js).

When moving to RestAPI, it is important to remember that the RestAPI is the public interface rather than the website content. The JSON model is **fully visible** to users. 
For example, if we need to display user profiles, password masking should be done on the JSON object rather than in the template. In order to do this, sometimes we need to create DTO objects for our RestAPI. 
 
### Synchronizing states between the backend and your AngularJS application

Synchronizing states is something that needs to be managed when you are developing a client-server architecture. You will need to give some thought to how your application updates its state from the backend or refresh its view whenever some state changes.


### Authentication

Having your client-side code exposed to the public makes it even more important to think through how you would like to authenticate your users and maintain a session with your application. 

You can check Dave Syer's series of blogs on how to integrate AngularJS with Spring Security [here](https://spring.io/blog/2015/01/12/spring-and-angular-js-a-secure-single-page-application).

### Testing

AngularJS comes with the necessary tools to help you perform testing at all layers of your Javascript development from unit to functional testing. Planning how you test and perform builds incorporating those tests will determine the quality of your front end client. We use a maven plugin called `frontend-maven-plugin` to assist us in our build tests.

### Conclusion

Migrating to AngularJS from JSP may seem daunting but it can be very rewarding in the long run as it makes for a more maintainable and testable user interface. The trend towards client side rendered views also encourages building more responsive web applications that were previously hampered by the design in server side rendering. The advent of HTML 5 and CSS3 has ushered us to a new era in View rendering technologies. The future of View layer development is on the client side and the future is now.