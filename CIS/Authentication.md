# Authentication

Types of users which are accessing the cluster
Admin, Developer, Bot, EndUser

EndUser -> Security of the endUser whos going to access the application which is deployed in the cluster is managed my the application themselves internally.

In the K8s we cannot create and view the list of users in the kubernetes where as the Service account we can create by `k create sa <sa-name>`
Note: User = Admin & Developers
      Bot = Service Account


