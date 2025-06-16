# ABDI - App Business Data Infrastructure

#### Description

This is an overview of advanced rails architecture inspired by various patterns and frameworks like Domain Driven Design, Repository patterns, Trailblazer, Dry.rb and others.

#### Key concepts

Everything revolves around 4 nodes: App, Business, Data, Infrastructure. Each node has certain responsibilities that must be not used by other nodes.

![ABDI.png](https://gist.github.com/assets/15248553/6a667bd7-0dae-4478-9177-9861852f1914)

### App

![app.png](https://gist.github.com/assets/15248553/a190e4ec-c6a9-4e27-b4a1-4ce2d4d96d3d)

##### Responsibilities

1. Representation
    - datatables
    - decorators
    - facades
    - helpers
    - mailers
    - serializers
    - view

2. Request input/output
    - channels (web-sockets)
    - controllers
    - services for AMQP communication
    - any other kind of external communications

3. No business logic  

### Business

[![business.png](https://gist.github.com/assets/15248553/cee33e58-8f8d-470c-9e9d-dfb8a6789e12)

##### Responsibilities

1. Use case validation  
1. Business rules execution  
1. Business rules callbacks
1. All business logic goes here

Business node is responsible for implementing client's requests (client is the one who pays for your code).
Business node concept is similar to Sagas, Use cases or  User stories, but is strict and simple.
It's structure should match business processes and real life domains. This is the only reason why application should be developed at all.  

Business node consist of:  
**Operations** - human-readable class that describes and performs business requirements. It's an entry point of business logic.  
**Actions** - are bricks for *Operations*. Could be called only within an *Operation*.  
**Forms** - data validation layer. First step of *Operation* call. *Operation* must not be executed if input data is invalid.
Business node could contain any custom components that are required for business requirements satisfaction.

The namespace/naming patter is: `BusinessDomainInPlural::Operations/Actions/Forms/::ClassNameAsVerb`

Structure:
```sh             
├── orders
│   ├── operations
│   │   ├── create.rb # -> calls Orders::Forms::Create, Orders::Actions::CalculatePrice, Orders::Actions::NotifyCustomer
│   │   └── cancel.rb
│   ├── forms
│   │   └── cancel.rb
│   └── actions
│       ├── calculate_price.rb
│       └── notify_customer.rb
└── Products
   ├── operations
   │   ├── create.rb
   │   └── delete.rb
   └── forms
       └── create.rb     
```

### Data

[![data.png](https://gist.github.com/assets/15248553/fd5e9b4a-b309-4e2b-a246-727e31e5d62d)


##### Responsibilities

1. Database read  
1. Database write
1. No business logic

This node is responsible for database communication. In other words - database read/write.  
*Data* is actually extracted and renamed rails models folder, BUT IT'S NOT behave like models.  

The key difference are:   
* zero business logic
* no callbacks except system required, like: cache clearing, data normalization, etc. 
* only persistence validation for correct DB usage, like: validate latitude if longitude present and vice-versa, validate email format and uniqueness, validate URL format, etc.  
no business validation like: customer region validation, age validation, etc. 

### Infrastructure

![insfrastructure.pnd](https://gist.github.com/assets/15248553/22190a41-0457-4644-af83-add96ac9c20f)


##### Responsibilities

1. Keep base classes: BaseOperation, BaseAction, etc.
1. Could keep clients for third-party API
1. Could keep any other custom non business constants and classes.
1. No business logic

[architecture_png]: ABDI.png
[app_png]: app.png
[business_png]: business.png
[data_png]: data.png
[infrastructure_png]: insfrastructure.png