# TrackingTime API — Verbatim export

> **Source:** Postman / developer documentation export (saved March 2026).  
> **Official:** [developers.trackingtime.co](https://developers.trackingtime.co/)  
> **Base URL:** `https://api.trackingtime.co/api/v4`

The content below is the full text that was provided for this project (chat preamble removed).

---

TrackingTime API
The TrackingTime API allows you to integrate our features and your account data into your own applications.

Making a Request
All request URLs start with this base URL. Requests must be sent via SSL. The path is prefixed with the API version. If we change the API in backward-incompatible ways, we'll bump the version marker and maintain stable support for the old URLs.

https://api.trackingtime.co/api/v4

For instance, to list all your workspace users, you'd append the users' endpoint path to the base url to get this:

https://api.trackingtime.co/api/v4/users

In case you belong to multiple workspaces, you will need to append the account id of the workspace you'd like to use as follows. If you don't specify an account id, the id of your default account will be used.

https://api.trackingtime.co/api/v4/:account_id/:endpoint_path, e.g.

https://api.trackingtime.co/api/v4/12345/users

You can find your account id by sending a request to the login endpoint. 

Authentication
To hit the ground running with the API, you can use HTTP Basic Authentication with your own email and password. This is secure since all requests use SSL.

Instead of using your personal password, you can also work with App Passwords, which are more flexible and allow you to use different passwords for different integrations you build.

View More
Plain Text
curl --location --request GET 'https://api.trackingtime.co/api/v4/1725/customers/372222?include_custom_fields=true' \
--header 'Authorization: {{vault:basic-auth}}'
Supplying basic auth headers
You need to construct and send basic auth headers with your requests, specifying your email (username) and password. To do this you need to perform the following steps:

Build a string of the form email:password, e.g. "john@doe.com:MyPassWord".

BASE64 encode that string.

Supply an Authorization header with content "Basic" followed by the encoded string. For example, the string "john@doe.com:MyPassWord" encodes to ImpvaG5AZG9lLmNvbTpNeVBhc3NXb3JkIg== in Base64, so you would make the request as follows:

Plain Text
curl -D- \
   -X GET \
   -H "Authorization: {{vault:basic-auth}}" \
   -H "Content-Type: application/json" \
   "https://api.trackingtime.co/api/v4/users"
Identifying your App
Your requests must include a User-Agent header with the name of your application and a link to your website or your email address so that we can reach out to you. You might also want to include your app's version.

View More
Plain Text
User-Agent: 'MyApp, Inc. (http://myapp.com/contact)'
User-Agent: 'MyApp v1.2 (email@yourapp.com)'
curl -u username:password -H User-Agent: 'MyApp (yourname@example.com)' https://app.trackingtime.co/api/v4/tasks
Standard JSON Response
All API endpoints return a JSON object with the following format:

json
{
    "response": {
        "status": 200,
        "version": "4.0",
        "message": "ok",
        "note": null,
        "note_type": null
    },
    "data": {
    }
}
These are the standard response parameters for all API calls.

version: The API version that handled the request.

status: The HTTP response status code.

message: The server response message, if any.

notes: Contains additional information for the end user.

note_type: Used to identify the type of note that should be shown to the user. [INFO | WARNING | ERROR | ALERT]

In case an error occurs, the response status will be set to 500 and you'll get an error description in the response message. Callers should always check the value of the status parameter in the response.

View More
json
{
    "response": {
        "status": 500,
        "version": "4.0",
        "note": null,
        "note_type": null,
        "message": "There is already another customer with that name. Please choose a different one."
    },
    "data": {
    }
}
© 2012-2023 TRACKING TIME, LLC. All rights reserved. Contact Us

AUTHORIZATION
Basic Auth
Username
<username>

Password
<password>

GET
Get a Customer
https://api.trackingtime.co/api/v4/:account_id/customers/:customer_id
Retrieves a single customer identified by its unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
include_custom_fields
<boolean>

Set to true to include custom fields in the response. Default: false.

PATH VARIABLES
account_id
<integer>

The account id.

customer_id
<integer>

The customer id.

Example Request
Get Customer
curl
curl --location 'https://api.trackingtime.co/api/v4/1725/customers/437855' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "name": "Sample Customer",
    "notes": null,
    "contact": null,
    "is_archived": false,
    "worked_hours": 0,
    "loc_worked_hours": "00:00",
    "total_projects": 0,
    "archived_projects": 0,
    "active_projects": 0,
    "average_hourly_rate": null,
    "worked_hours_today": 0,
    "worked_hours_this_week": 0,
    "worked_hours_this_month": 0,
    "loc_worked_hours_today": "00:00",
    "loc_worked_hours_this_week": "00:00",
    "loc_worked_hours_this_month": "00:00",
    "color_index": 144,
    "id": 123456,
    "created_at": "2023-03-07 14:51:46",
    "updated_at": null,
    "json": null
  }
}
GET
List Customers
https://api.trackingtime.co/api/v4/:account_id/customers
Retrieves a list of customers using the supported input parameters for filtering results.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
ClientApp
TrackingTime Mac

PARAMS
include_custom_fields
<boolean>

Set to true to include to include custom fields in the response. Default: false.

PATH VARIABLES
account_id
<integer>

The account id.

Example Request
List Customers
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/customers/?include_custom_fields=%3Cboolean%3E' \
--header 'ClientApp;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "name": "Sample Customer",
      "notes": null,
      "contact": null,
      "is_archived": false,
      "worked_hours": 0,
      "loc_worked_hours": "00:00",
      "total_projects": 0,
      "archived_projects": 0,
      "active_projects": 0,
      "average_hourly_rate": null,
      "worked_hours_today": 0,
      "worked_hours_this_week": 0,
      "worked_hours_this_month": 0,
      "loc_worked_hours_today": "00:00",
      "loc_worked_hours_this_week": "00:00",
      "loc_worked_hours_this_month": "00:00",
      "color_index": 144,
      "id": 123456,
      "created_at": "2023-03-07 14:51:46",
      "updated_at": null,
      "json": null
    }
  ]
}
POST
Update a Customer
https://api.trackingtime.co/api/v4/:account_id/customers/update/:customer_id
Updates a single customer identified by its unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
name
<string>

Human-friendly identifiert. Unique.

custom_fields
<json>

Supported format: [{"id":,"value":"value"}]

PATH VARIABLES
account_id
<integer>

The account id.

customer_id
<integer>

The customer id.

Example Request
Update Customer
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/customers/update/<integer>?name=' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "name": "Sample Customer",
    "notes": null,
    "contact": null,
    "is_archived": false,
    "worked_hours": 0,
    "loc_worked_hours": "00:00",
    "total_projects": 0,
    "archived_projects": 0,
    "active_projects": 0,
    "average_hourly_rate": null,
    "worked_hours_today": 0,
    "worked_hours_this_week": 0,
    "worked_hours_this_month": 0,
    "loc_worked_hours_today": "00:00",
    "loc_worked_hours_this_week": "00:00",
    "loc_worked_hours_this_month": "00:00",
    "color_index": 144,
    "id": 123456,
    "created_at": "2023-03-07 14:51:46",
    "updated_at": null,
    "json": null
  }
}
GET
Delete a Customer
https://api.trackingtime.co/api/v4/:account_id/customers/delete/:customer_id
⛔️ WARNING: This action cannot be undone.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PATH VARIABLES
account_id
<integer>

The account id.

customer_id
<integer>

The id of the customer to be deleted.

Example Request
Delete Customer
curl
curl --location 'https://api.trackingtime.co/api/v4/customers/delete/<integer>' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {}
}
Projects
This section describes how to add, edit and retrieve your workspace's projects.

The Projects API will be useful if you'd like to import projects from apps like Jira, Asana or MS Planner into TrackingTime.

AUTHORIZATION
Basic Auth
This folder is using Basic Auth from collectionTrackingTime API
GET
Get a Project
https://api.trackingtime.co/api/v4/:account_id/projects/:project_id/min
Retrieves a single project identified by its unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
include_tasks
<boolean>

Set to true to include all project tasks in the response.

include_task_lists
<boolean>

Set to true to include all project task lists in the response.

include_custom_fields
<boolean>

Set to true to include custom fields in the response.

include_closed_tasks
<boolean>

Set to true to include project tasks marked as done in the response.

PATH VARIABLES
account_id
<integer>

The account id.

project_id
<integer>

The project id.

Example Request
Get Project
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/<integer>/projects/<integer>/min' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "start_date": null,
    "loc_start_date": null,
    "end_date": null,
    "loc_end_date": null,
    "delivery_date": null,
    "loc_delivery_date": null,
    "estimated_time": 0,
    "loc_estimated_time": "00:00",
    "accumulated_time": 0,
    "is_archived": false,
    "following": false,
    "notes": null,
    "name": "Sample Project",
    "color": null,
    "tasks": null,
    "task_lists": null,
    "customer": null,
    "service": null,
    "billing": {
      "is_billable": true,
      "hourly_rate": null,
      "loc_hourly_rate": null,
      "hourly_cost": null,
      "loc_hourly_cost": null,
      "fixed_rate": null,
      "loc_fixed_rate": null,
      "billable_hours": 0,
      "non_billable_hours": 0,
      "loc_billable_hours": "0",
      "loc_non_billable_hours": "0"
    },
    "is_public": false,
    "default_view": null,
    "worked_hours": 0,
    "loc_worked_hours": "00:00",
    "worked_hours_today": null,
    "worked_hours_this_week": null,
    "worked_hours_this_month": null,
    "loc_worked_hours_today": null,
    "loc_worked_hours_this_week": null,
    "loc_worked_hours_this_month": null,
    "active_tasks": 0,
    "archived_tasks": 0,
    "is_template": false,
    "users_hours": null,
    "coworkers": null,
    "files": null,
    "account_index": null,
    "user_preferences": null,
    "doc_id": 12345,
    "third_party_data": null,
    "user_index": null,
    "id": 1,
    "created_at": "2023-03-07 16:33:30",
    "updated_at": null,
    "json": null
  }
}
GET
List Projects
https://api.trackingtime.co/api/v4/:account_id/projects
Retrieves a list of projects using the supported input parameters for filtering results.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
filter
<string>

[ALL | ACTIVE | ARCHIVED | FOLLOWING]. Default = ACTIVE

include_custom_fields
<boolean>

Set to 'true' to include custom fields in the response. Default = false.

PATH VARIABLES
account_id
<integer>

The account id.

Example Request
List Projects
curl
curl --location 'https://api.trackingtime.co/api/v4/:account_id/projects/min' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "start_date": null,
      "loc_start_date": null,
      "end_date": null,
      "loc_end_date": null,
      "delivery_date": null,
      "loc_delivery_date": null,
      "estimated_time": 0,
      "loc_estimated_time": "00:00",
      "accumulated_time": 0,
      "is_archived": false,
      "following": false,
      "notes": null,
      "name": "Sample Project",
      "color": null,
      "tasks": null,
      "task_lists": null,
      "customer": null,
      "service": null,
      "billing": {
        "is_billable": true,
        "hourly_rate": null,
        "loc_hourly_rate": null,
        "hourly_cost": null,
        "loc_hourly_cost": null,
        "fixed_rate": null,
        "loc_fixed_rate": null,
        "billable_hours": 0,
        "non_billable_hours": 0,
        "loc_billable_hours": "0",
        "loc_non_billable_hours": "0"
      },
      "is_public": false,
      "default_view": null,
      "worked_hours": 0,
      "loc_worked_hours": "00:00",
      "worked_hours_today": null,
      "worked_hours_this_week": null,
      "worked_hours_this_month": null,
      "loc_worked_hours_today": null,
      "loc_worked_hours_this_week": null,
      "loc_worked_hours_this_month": null,
      "active_tasks": 0,
      "archived_tasks": 0,
      "is_template": false,
      "users_hours": null,
      "coworkers": null,
      "files": null,
      "account_index": null,
      "user_preferences": null,
      "doc_id": 12345,
      "third_party_data": null,
      "user_index": null,
      "id": 1,
      "created_at": "2023-03-07 16:33:30",
      "updated_at": null,
      "json": null
    }
  ]
}
GET
List Project Users
https://api.trackingtime.co/api/v4/:account_id/projects/:project_id/users
List all the users assigned to a project, i.e. they have been assigned at least one project task.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PATH VARIABLES
account_id
<integer>

The account id.

project_id
<integer>

The project id.

Example Request
List Project Users
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/:account_id/projects/<integer>/users' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "name": "John",
      "surname": "Doe",
      "avatar_url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/0dc5743c1798db6fefff7f90c59c3d47_7k8lnp181mj23vtpasdfasf3j8enkjkdf.jpg",
      "token": null,
      "third_party_data": null,
      "id": 12345678
    },
    {
      "name": "Alice",
      "surname": "Barazal",
      "avatar_url": "https://www.gravatar.com/avatar/53abcd46829e7e4809d0cafasdf3bdfdc316f5.jpg?d=null",
      "token": null,
      "third_party_data": null,
      "id": 542414123
    }
  ]
}
POST
Update a Project
https://api.trackingtime.co/api/v4/:account_id/projects/update/:project_id
Updates a single project identified by its unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
name
<string>

is_billable
<boolean>

hourly_rate
<integer>

fixed_rate
<integer>

is_public
<boolean>

default_view
<string>

custom_fields
<json>

Supported format: [{"id":,"value":"value"}]

customer_id
<integer>

Link a customer to this project by providing the customer id.

customer_name
<string>

Link a customer to this project by providing the customer name. If the customer doesn't exist yet, it will be automatically created.

estimated_time
<double>

The estimated duration (in hours) required to complete the project.

PATH VARIABLES
account_id
<integer>

The account id.

project_id
<integer>

The project id.

Example Request
Update Project
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/:account_id/projects/update/<integer>' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (15)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "start_date": null,
    "loc_start_date": null,
    "end_date": null,
    "loc_end_date": null,
    "delivery_date": null,
    "loc_delivery_date": null,
    "estimated_time": 0,
    "loc_estimated_time": "00:00",
    "accumulated_time": 0,
    "is_archived": false,
    "following": false,
    "notes": null,
    "name": "Sample Project",
    "color": null,
    "tasks": null,
    "task_lists": null,
    "customer": null,
    "service": null,
    "billing": {
      "is_billable": true,
      "hourly_rate": null,
      "loc_hourly_rate": null,
      "hourly_cost": null,
      "loc_hourly_cost": null,
      "fixed_rate": null,
      "loc_fixed_rate": null,
      "billable_hours": 0,
      "non_billable_hours": 0,
      "loc_billable_hours": "0",
      "loc_non_billable_hours": "0"
    },
    "is_public": false,
    "default_view": null,
    "worked_hours": 0,
    "loc_worked_hours": "00:00",
    "worked_hours_today": null,
    "worked_hours_this_week": null,
    "worked_hours_this_month": null,
    "loc_worked_hours_today": null,
    "loc_worked_hours_this_week": null,
    "loc_worked_hours_this_month": null,
    "active_tasks": 0,
    "archived_tasks": 0,
    "is_template": false,
    "users_hours": null,
    "coworkers": null,
    "files": null,
    "account_index": null,
    "user_preferences": null,
    "doc_id": 12345,
    "third_party_data": null,
    "user_index": null,
    "id": 1,
    "created_at": "2023-03-07 16:33:30",
    "updated_at": null,
    "json": null
  }
}
GET
Close a Project
https://api.trackingtime.co/api/v4/:account_id/projects/close/:project_id
Archives a single project identified by its unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PATH VARIABLES
account_id
<integer>

The account id.

project_id
<integer>

The project id.

Example Request
Close Project
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/:account_id/projects/close/1542214' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "start_date": null,
    "loc_start_date": null,
    "end_date": null,
    "loc_end_date": null,
    "delivery_date": null,
    "loc_delivery_date": null,
    "estimated_time": 0,
    "loc_estimated_time": "00:00",
    "accumulated_time": 0,
    "is_archived": false,
    "following": false,
    "notes": null,
    "name": "Sample Project",
    "color": null,
    "tasks": null,
    "task_lists": null,
    "customer": null,
    "service": null,
    "billing": {
      "is_billable": true,
      "hourly_rate": null,
      "loc_hourly_rate": null,
      "hourly_cost": null,
      "loc_hourly_cost": null,
      "fixed_rate": null,
      "loc_fixed_rate": null,
      "billable_hours": 0,
      "non_billable_hours": 0,
      "loc_billable_hours": "0",
      "loc_non_billable_hours": "0"
    },
    "is_public": false,
    "default_view": null,
    "worked_hours": 0,
    "loc_worked_hours": "00:00",
    "worked_hours_today": null,
    "worked_hours_this_week": null,
    "worked_hours_this_month": null,
    "loc_worked_hours_today": null,
    "loc_worked_hours_this_week": null,
    "loc_worked_hours_this_month": null,
    "active_tasks": 0,
    "archived_tasks": 0,
    "is_template": false,
    "users_hours": null,
    "coworkers": null,
    "files": null,
    "account_index": null,
    "user_preferences": null,
    "doc_id": 12345,
    "third_party_data": null,
    "user_index": null,
    "id": 1,
    "created_at": "2023-03-07 16:33:30",
    "updated_at": null,
    "json": null
  }
}
GET
Open a Project
https://api.trackingtime.co/api/v4/:account_id/projects/open/:project_id
Re-activates a single project identified by its unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PATH VARIABLES
account_id
<integer>

The account id.

project_id
<integer>

The project id.

Example Request
Open Project
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/:account_id/projects/open/<integer>' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "start_date": null,
    "loc_start_date": null,
    "end_date": null,
    "loc_end_date": null,
    "delivery_date": null,
    "loc_delivery_date": null,
    "estimated_time": 0,
    "loc_estimated_time": "00:00",
    "accumulated_time": 0,
    "is_archived": false,
    "following": false,
    "notes": null,
    "name": "Sample Project",
    "color": null,
    "tasks": null,
    "task_lists": null,
    "customer": null,
    "service": null,
    "billing": {
      "is_billable": true,
      "hourly_rate": null,
      "loc_hourly_rate": null,
      "hourly_cost": null,
      "loc_hourly_cost": null,
      "fixed_rate": null,
      "loc_fixed_rate": null,
      "billable_hours": 0,
      "non_billable_hours": 0,
      "loc_billable_hours": "0",
      "loc_non_billable_hours": "0"
    },
    "is_public": false,
    "default_view": null,
    "worked_hours": 0,
    "loc_worked_hours": "00:00",
    "worked_hours_today": null,
    "worked_hours_this_week": null,
    "worked_hours_this_month": null,
    "loc_worked_hours_today": null,
    "loc_worked_hours_this_week": null,
    "loc_worked_hours_this_month": null,
    "active_tasks": 0,
    "archived_tasks": 0,
    "is_template": false,
    "users_hours": null,
    "coworkers": null,
    "files": null,
    "account_index": null,
    "user_preferences": null,
    "doc_id": 12345,
    "third_party_data": null,
    "user_index": null,
    "id": 1,
    "created_at": "2023-03-07 16:33:30",
    "updated_at": null,
    "json": null
  }
}
GET
Get a Task
https://api.trackingtime.co/api/v4/:account_id/tasks/:task_id
Retrieves a single task identified by its unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
include_custom_fields
<boolean>

Set to true to include custom fields in the response.

PATH VARIABLES
account_id
<integer>

The account id.

task_id
<integer>

The task id.

Example Request
Get Task
curl
curl --location 'https://api.trackingtime.co/api/v4/<integer>/tasks/<integer>' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "name": "Sample Task",
    "type": "PERSONAL",
    "visibility": null,
    "project_id": 1266926,
    "project": "PROJECT",
    "project_accumulated_time": 11103,
    "project_loc_worked_hours": "03:05",
    "project_is_public": true,
    "project_is_favorite": true,
    "customer": null,
    "customer_id": null,
    "service": null,
    "service_id": null,
    "estimated_time": 0,
    "loc_estimated_time": "00:00",
    "accumulated_time": 0,
    "is_archived": false,
    "start_date": "2023-03-08 10:45:22",
    "loc_start_date": "08.03.2023 10:45:22",
    "end_date": null,
    "loc_end_date": null,
    "due_date": null,
    "loc_due_date": null,
    "list_position": 0,
    "list_id": null,
    "list_name": null,
    "worked_hours": 0,
    "loc_worked_hours": "00:00",
    "accumulated_time_display": "00:00:00",
    "tracking": false,
    "account_id": 1725,
    "color": null,
    "day_index": 0,
    "index": 0,
    "skill": null,
    "user": {
      "name": "John",
      "surname": " Doe",
      "avatar_url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/40a7f9ee025e86b2e376a8bd37cb3467_ng03saghs98bnaod2rvcdi6vqb232.jpg",
      "token": "CB3UT2IJJSH5CU4A9S0PK7MLAFAASDFAASSK3",
      "third_party_data": null,
      "id": 1
    },
    "billing": {
      "is_billable": true,
      "hourly_rate": null,
      "loc_hourly_rate": null,
      "hourly_cost": null,
      "loc_hourly_cost": null,
      "fixed_rate": null,
      "loc_fixed_rate": null,
      "billable_hours": null,
      "non_billable_hours": null,
      "loc_billable_hours": null,
      "loc_non_billable_hours": null
    },
    "project_billing": null,
    "service_item": null,
    "tracking_event": null,
    "subtasks": [],
    "comments": [],
    "users": [],
    "files": null,
    "description": null,
    "created_by": {
      "name": "John",
      "surname": " Doe",
      "avatar_url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/40a7f9ee025e86b2e376a8bd37cb3467_ng03saghs98bnaod2sdfafrvcdi6vqb.jpg",
      "token": "CB3UT2IJJSH5CU4A9S0PK7MLAFAASDFAASSK3",
      "third_party_data": null,
      "id": 1
    },
    "task_permalink": null,
    "third_party_data": null,
    "id": 94656212106,
    "created_at": "2023-03-08 10:45:22",
    "updated_at": null,
    "json": null
  },
  "set_custom_fields": [],
  "available_custom_fields": [],
  "users": {}
}
GET
List Tasks
https://api.trackingtime.co/api/v4/tasks/paginated
Retrieves a list of tasks using the supported input parameters for filtering results.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
filter
<string>

Supported values: [ALL | ACTIVE | ARCHIVED | TRACKING]. Use the TRACKING filter to retrieve all account tasks currently tracking. Default = ACTIVE.

page
<integer>

Use for pagination. Default:

page_size
<integer>

Use for pagination. Default:

include_custom_fields
<boolean>

Set to true to include custom fields in the response.

Example Request
List Tasks
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/tasks/paginated?page=0&page_size=2' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "name": "Sample Task",
      "type": "PERSONAL",
      "visibility": null,
      "project_id": 1266926,
      "project": "PROJECT",
      "project_accumulated_time": 11103,
      "project_loc_worked_hours": "03:05",
      "project_is_public": true,
      "project_is_favorite": true,
      "customer": null,
      "customer_id": null,
      "service": null,
      "service_id": null,
      "estimated_time": 0,
      "loc_estimated_time": "00:00",
      "accumulated_time": 0,
      "is_archived": false,
      "start_date": "2023-03-08 10:45:22",
      "loc_start_date": "08.03.2023 10:45:22",
      "end_date": null,
      "loc_end_date": null,
      "due_date": null,
      "loc_due_date": null,
      "list_position": 0,
      "list_id": null,
      "list_name": null,
      "worked_hours": 0,
      "loc_worked_hours": "00:00",
      "accumulated_time_display": "00:00:00",
      "tracking": false,
      "account_id": 1725,
      "color": null,
      "day_index": 0,
      "index": 0,
      "skill": null,
      "user": {
        "name": "John",
        "surname": " Doe",
        "avatar_url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/40a7f9ee025e86b2e376a8bd37cb3467_ng03saghs98bnaod2rvcdi6vqb232.jpg",
        "token": "CB3UT2IJJSH5CU4A9S0PK7MLAFAASDFAASSK3",
        "third_party_data": null,
        "id": 1
      },
      "billing": {
        "is_billable": true,
        "hourly_rate": null,
        "loc_hourly_rate": null,
        "hourly_cost": null,
        "loc_hourly_cost": null,
        "fixed_rate": null,
        "loc_fixed_rate": null,
        "billable_hours": null,
        "non_billable_hours": null,
        "loc_billable_hours": null,
        "loc_non_billable_hours": null
      },
      "project_billing": null,
      "service_item": null,
      "tracking_event": null,
      "subtasks": [],
      "comments": [],
      "users": [],
      "files": null,
      "description": null,
      "created_by": {
        "name": "John",
        "surname": " Doe",
        "avatar_url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/40a7f9ee025e86b2e376a8bd37cb3467_ng03saghs98bnaod2sdfafrvcdi6vqb.jpg",
        "token": "CB3UT2IJJSH5CU4A9S0PK7MLAFAASDFAASSK3",
        "third_party_data": null,
        "id": 1
      },
      "task_permalink": null,
      "third_party_data": null,
      "id": 94656212106,
      "created_at": "2023-03-08 10:45:22",
      "updated_at": null,
      "json": null
    }
  ],
  "set_custom_fields": [],
  "available_custom_fields": [],
  "users": {}
}
GET
Delete a Task
https://api.trackingtime.co/api/v4/:account_id/tasks/delete/:task_id
⛔️ WARNING: This action cannot be undone. When deleting a task, all events associated to it will also be deleted. Use this endpoint with extreme caution!

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
delete_all
false

PATH VARIABLES
account_id
<integer>

The account id.

task_id
<integer>

The task id.

Example Request
Delete Task
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/<integer>/tasks/delete/<integer>' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {}
}
Events
This section describes how to add, edit and retrieve events logged by you and the rest of your team.

An event represents a specific period in time, in which a user worked on a given task or project and is a core entity in TrackingTime's data model.

Events must always have a start and end date and are always associated to a single person within your workspace.

Additionally, events can be associated to a task, which in turns can be associated to a project and customer.

This flexible data model allows you and your team to categorize working time spent at your company in different ways depending on your specific organizational and reporting needs.

Parameter Abbreviations
The List Events endpoint can return very large datasets, depending on the time frame and page size you request, the number of users in your account, as well as other factors.

In order to improve performance and reduce response size, we use parameter abbreviations instead of the regular parameter names.

Here's the list of abbreviations for your reference.

a: is archived
b: is billable?
bd: is billed?
c: customer name
cci: customer color index
cid: customer id
co: project color
d: duration
dd: due date
e: end date
er: event repeat
ety: eventy type
et: task estimated time
g: user groups
id: event id
n: event notes
p: project name
pif: project is favorite
pid: project id
pn: project notes
r: hourly rate
rf: repeat frequency
re: repeat every
rgi: repeat group id
s: start date
sci: service color index
se: service name
seid: service id
ts: tags
t: task name
tid: task id
tl: task list name
to: token for requesting the user's avatar URL
tt: task type
tv: task visibility
tz: timezone
u: user name
uau: user avatar URL
uci: user color index
uid: user id
uhr: user hourly rate
uhc: user hourly cost
AUTHORIZATION
Basic Auth
This folder is using Basic Auth from collectionTrackingTime API
GET
List Events
https://api.trackingtime.co/api/v4/:account_id/events
Retrieves a list of events using the supported input parameters for filtering results.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
filter
<string>

Optional. Filter events by entity. Supported values: [USER | PROJECT | CUSTOMER | TASK | COMPANY]. Default: USER.

id
<integer>

Optional. The id of the entity you specifed for the parameter filter. Required when using a filter that's not COMPANY.

from
<date>

Required. Specify the start date for a period to retrieve events.

to
<date>

Required. Specify the end date a period to retrieve events.

page
<integer>

Optional. Used for pagination. Default: 0.

page_size
<integer>

Optional. Used for pagination. Default: 1,000. Max: 20,000.

order
<string>

Optional. Supported values: [asc | desc]. Indicate how events should be ordered by start date.

include_custom_fields
<boolean>

Set to true to include custom fields in the response. Default: false.

sort_by
<string>

Optional. Accepted values: [created_at | updated_at]. If provided, the from and to parameters will filter events based on their creation or last update date -- instead of the events' start date.

PATH VARIABLES
account_id
<integer>

The account id.

Example Request
List Events
curl
curl --location 'https://api.trackingtime.co/api/v4/1725/events/min?id=542588' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (14)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "id": 3490132317,
      "s": "2023-05-11 10:00:00",
      "e": "2023-05-11 12:00:00",
      "d": 7202320,
      "se": null,
      "seid": null,
      "c": null,
      "cid": null,
      "p": "Sample Project",
      "pid": 15422323176,
      "pn": "",
      "pif": false,
      "t": "Sample Task",
      "tid": 9462325383,
      "tt": "PERSONAL",
      "tv": null,
      "tl": null,
      "dd": null,
      "et": null,
      "a": false,
      "u": "John Doe",
      "uid": 542588,
      "uhr": 0,
      "uhc": 0,
      "uau": "https://lh3.googleusercontent.com/a/AEdFTp7KxvqBPlJV1vPt3VXesOasdfasSDSAV2biHZQXc3-HlRlzvB=s96-c",
      "tz": "GMT-03:00",
      "n": "Sample Notes",
      "co": "4",
      "r": null,
      "b": true,
      "bd": false,
      "to": "h04tu63ndd8hfvtaasdfasdfbi3stj0nf8",
      "er": "CUSTOM",
      "rf": "DAILY",
      "re": 1,
      "rgi": 34901248,
      "ety": "WORK",
      "uci": 107,
      "cci": null,
      "sci": null,
      "has_files": false,
      "toi": null,
      "users": null
    }
  ],
  "set_custom_fields": [],
  "available_custom_fields": [],
  "users": {},
  "schedule_assignments": {}
}
GET
List Flat Events
https://api.trackingtime.co/api/v4/:account_id/events/flat
Return the list of events according to the received filter, adding flattened custom field data to each event.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
filter
<string>

Optional. Filter events by entity. Supported values: [USER | PROJECT | CUSTOMER | TASK | COMPANY]. Default: USER.

id
<integer>

Optional. The id of the entity you specifed for the parameter filter. Required when using a filter that's not COMPANY.

from
<date>

Required. Specify the start date for a period to retrieve events.

to
<date>

Required. Specify the end date a period to retrieve events.

page
<integer>

Optional. Used for pagination. Default: 0.

page_size
<integer>

Optional. Used for pagination. Default: 1,000. Max: 5,000.

include_custom_fields
<boolean>

Set to true to include custom fields in the response. Default: false.

PATH VARIABLES
account_id
<integer>

The account id.

Example Request
List Flat Events
curl
curl --location 'https://api.trackingtime.co/api/v4/1725/events/min?id=542588' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (14)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "ID": 22176912,
      "Start": "2019-01-03 09:11:38",
      "End": "2019-01-03 10:31:38",
      "Duration": 45,
      "Service": null,
      "Service Id": null,
      "Customer": "Correntoso",
      "Customer Id": 372222,
      "Project": "Dev",
      "Project Id": 813712,
      "Project Notes": null,
      "Project is favorite": false,
      "Task": "Daily support",
      "Task Id": 4539199,
      "Task Type": "PERSONAL",
      "Task Visibility": "",
      "Task List": null,
      "Due Date": null,
      "Estimated Time": 0,
      "Is Archived": true,
      "User": "Diego",
      "User Id": 79634,
      "User Hourly Rate": 50,
      "User Hourly Cost": 100,
      "User Avatar Url": "https://lh3.googleusercontent.com/a/ACg8ocKvAjq",
      "Timezone": "GMT-03:00",
      "Notes": null,
      "Color": "7",
      "Hourly Rate": null,
      "Is Billable": true,
      "Is Billed": false,
      "Icalendar Token": "CB3UT2IJJSH5CU4A9",
      "Event Repeat": null,
      "Repeat Rrequency": null,
      "Repeat Every": null,
      "Repeat Group Id": null,
      "Event Type": "WORK",
      "User Color Index": 2,
      "Customer Color Index": 16,
      "Service Color Index": null,
      "Has files": false,
      "Time Off Id": null,
      "Users": null,
      "Currency (Customer CF)": null,
      "Teams (Project CF)": null,
      "Locations (Event CF)": null,
      "Country (User CF)": null,
      "INVOICE_ID (Event CF)": null,
      "Start Date (User CF)": "2013-07-07",
      "PROJECT_TIMELINE_ROW (Project CF)": null,
      "PROJECT_TIMELINE_START (Project CF)": null,
      "Start Date (Project CF)": null,
      "Third Party Service (Project CF)": null,
      "Third Party ID (Project CF)": null,
      "Third Party Service (Event CF)": null,
      "Third Party ID (Event CF)": null,
      "Start Date a (Project CF)": null,
      "Start Date b (Project CF)": null
    },
    {
      "ID": 22187704,
      "Start": "2019-01-03 13:00:00",
      "End": "2019-01-03 16:15:00",
      "Duration": 11700,
      "Service": null,
      "Service Id": null,
      "Customer": null,
      "Customer Id": null,
      "Project": "Management",
      "Project Id": 32323232,
      "Project Notes": null,
      "Project is favorite": false,
      "Task": "Office time",
      "Task Id": 4608311,
      "Task Type": "PERSONAL",
      "Task Visibility": "",
      "Task List": "Marketing",
      "Due Date": null,
      "Estimated Time": 0,
      "Is Archived": true,
      "User": "Diego",
      "User Id": 79634,
      "User Hourly Rate": 50,
      "User Hourly Cost": 100,
      "User Avatar Url": "https://lh3.googleusercontent.com/a/ACg8ocKvAjq36m06nwsr_dHy1naWE5zCW",
      "Timezone": "GMT-03:00",
      "Notes": null,
      "Color": "3",
      "Hourly Rate": null,
      "Is Billable": true,
      "Is Billed": false,
      "Icalendar Token": "RERERERRDFDFDFD54665456",
      "Event Repeat": null,
      "Repeat Rrequency": null,
      "Repeat Every": null,
      "Repeat Group Id": null,
      "Event Type": "WORK",
      "User Color Index": 2,
      "Customer Color Index": null,
      "Service Color Index": null,
      "Has files": false,
      "Time Off Id": null,
      "Users": null,
      "Currency (Customer CF)": null,
      "Teams (Project CF)": null,
      "Locations (Event CF)": null,
      "Country (User CF)": null,
      "INVOICE_ID (Event CF)": null,
      "Start Date (User CF)": "2013-07-07",
      "PROJECT_TIMELINE_ROW (Project CF)": null,
      "PROJECT_TIMELINE_START (Project CF)": null,
      "Start Date (Project CF)": null,
      "Third Party Service (Project CF)": null,
      "Third Party ID (Project CF)": null,
      "Third Party Service (Event CF)": null,
      "Third Party ID (Event CF)": null,
      "Start Date a (Project CF)": null,
      "Start Date b (Project CF)": null
    }
  ],
  "users": {},
  "schedule_assignments": {}
}
GET
List Filtered Events
https://api.trackingtime.co/api/v4/:account_id/events/filtered
This endpoint provides a flexible way to retrieve events using advanced filters and operators defined within a JSON-formatted query_data object. Unlike the standard List Events endpoint, this advanced query allows precise control over the data retrieved, including the ability to filter, sort, group, and format results according to your specific requirements.

Query Data Object
The query_data object is the core of this API request. It defines filters, operators, and other options for customizing the response. Each section of the JSON structure has a specific purpose, detailed below.

Sample query_data JSON Object
View More
json
{
    "date_range":{
        "start":null,
        "end":null,
        "period":"last_n_days",
    },
    "parameters": [
        "start",
        "end",
        "duration",
        "task_name"
    ],
    "filters": [
        {
            "by": "user",
            "operator": "is",
            "value": [
                "John Doe",
                "Alice Appleseed",
            ]
        },
        {
            "by": "project",
            "operator": "contains",
            "value": [
                "Project"
            ]
        }
    ],
    "sorts": [
        {
            "by": "date",
            "order": "asc"
        },
        {
            "by": "task",
            "order": "asc"
        }
    ],
    "groups": [
        {
            "by": "user",
            "order": "asc"
        },
        {
            "by": "project",
            "order": "asc"
        }
    ],
    "rounding": {
        "by":10,
        "method":"up"
    }
}
Parameters Description
The following paramteres are supported in the query_data object.

date_range

parameters

filters

sorts

groups

rounding

Date Range
Specifies the time period for filtering events. Supported values:

today

this_week

yesterday

last_week

this_month

last_month

last_n_days = Retrieves all events from the last n days, e.g. “last_20_days”. The max allowed number is 90 days.

all: Retrieves all events since the account’s creation day until now.

Response Parameters
Defines the fields to include in the API response. Specifying only the needed fields reduces response size and improves performance. Example:

json
["start", "end", "duration", "task_name"]
Filters
Filters refine the event selection based on specified criteria. Each filter object includes:

by: The field to filter by (e.g., user, project, task).

operator: The comparison operator.

value: The value(s) to filter agains (e.g.,["John Doe", "Project Alpha"].

Supported operators

is: Exact match for one or more values. For example, finding events where the user name exactly matches "John Smith".

is_not: Excludes exact matches for one or more values. For example, excluding events where the project name is "Internal".

contains: Partial match that checks if the field contains the specified text. For example, finding tasks containing the word "Meeting".

does_not_contain: Excludes items where the field contains the specified text. For example, excluding tasks containing the word "Break".

is_set: Checks if a field has any value (is not empty or null). Useful for finding events that have specific fields populated.

is_not_set: Checks if a field is empty or null. Useful for finding events missing certain information.

=: Numerical equality operator, used for comparing numeric values. For example, finding events with exactly 2 hours duration.

Sorts
Determines the order of results. Each sort object includes:

by: The field to sort by (e.g., date, task).

order: The sort direction. Options:

asc (Ascending): Sorts values from lowest to highest, or alphabetically from A to Z. For example, sorting dates will show the earliest entries first.

desc (Descending): Sorts values from highest to lowest, or alphabetically from Z to A. For example, sorting by duration will show the longest time entries first.

Groups
Groups events by specified fields and applies optional sorting within groups. Each group object includes:

• by: The field to group by (e.g., user, project).

• order: Sort direction within the group (asc or desc).

Rounding
Adjusts event durations to standard intervals. Useful for billing or standardization. Includes:

by: The rounding interval in minutes. Supported values: 1, 6, 10, 15, 30, 60.

method: The rounding method. Options:

round_up: Always rounds up to the next interval.

round: Rounds to the nearest interval.

round_down: Always rounds down to the previous interval.

Custom Field Parameters
Custom fields can be included in both response parameters and filters. Use the following format to reference custom fields:

Format: cf_{custom_field_type}_{object_id}

Example: cf_event_1234567

Supported Types

customer

project

task

user

event

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
page
<integer>

Optional. Used for pagination. Default: 0.

page_size
<integer>

Optional. Used for pagination. Default: 1,000. Max: 5,000.

query_data
<json>

A JSON object that allows you to provide a series of advanced filters and operators.

PATH VARIABLES
account_id
<integer>

The account id.

Example Request
List Filtered Events
View More
curl
curl --location -g 'https://api.trackingtime.co/api/v4/1725/events/min?query_data={%22date_range%22%3A{%22start%22%3Anull%2C%22end%22%3Anull%2C%22period%22%3A%22last_90_days%22}%2C%22parameters%22%3A[]%2C%22filters%22%3A[]%2C%22groups%22%3A[]}' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (14)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "id": 101446294,
      "s": "2025-01-15 18:00:00",
      "e": "2025-01-15 18:57:00",
      "d": 3420,
      "pid": 2010679,
      "pif": false,
      "tid": 13583209,
      "tt": "PERSONAL",
      "a": false,
      "uid": 413260,
      "tz": "GMT+01:00",
      "co": "3",
      "to": "lf5gvq2cbpbjuod748c2e6tkj7",
      "ety": "WORK",
      "uci": 47
    }
  ]
}
GET
Get a User
https://api.trackingtime.co/api/v4/:account_id/users/:user_id
Retrieves a single user identified by their unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
include_custom_fields
<boolean>

Set to true to include custom fields in the response. Default: false.

PATH VARIABLES
account_id
<integer>

The account id.

user_id
<integer>

The user id.

Example Request
Get User
curl
curl --location 'https://api.trackingtime.co/api/v4/<integer>/users/<integer>' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "account_id": 1,
    "name": "John",
    "surname": " Doe",
    "email": "johndoe@domain.com",
    "country": null,
    "loc_country": null,
    "role": "ADMIN",
    "avatar_url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/40a7f9ee025e86b2e376a8bd37cb3467_ng03saghs98bnaod2rvcdi6vqbfsdfssf.jpg",
    "is_archived": false,
    "is_freelance": false,
    "is_owner": true,
    "status": "VERIFIED",
    "token": null,
    "icalendar_feed": "https://app.trackingtime.co/v4/1725/users/79634/icalendar/CB3UT2IJJSH5CU4A9S0PK7MLK3DFSAF",
    "worked_hours_today": null,
    "worked_hours_this_week": null,
    "worked_hours_this_month": null,
    "loc_worked_hours_today": null,
    "loc_worked_hours_this_week": null,
    "loc_worked_hours_this_month": null,
    "identity_provider": null,
    "identity_providers": [],
    "worked_hours": null,
    "loc_worked_hours": null,
    "billing": {
      "is_billable": false,
      "hourly_rate": 50,
      "loc_hourly_rate": "50",
      "hourly_cost": 100,
      "loc_hourly_cost": "100",
      "fixed_rate": 0,
      "loc_fixed_rate": "0",
      "billable_hours": null,
      "non_billable_hours": null,
      "loc_billable_hours": null,
      "loc_non_billable_hours": null
    },
    "settings": {
      "show_message": false,
      "show_step_by_step": true,
      "language": "EN",
      "currency": "USD",
      "currency_name": "United States Dollars - USD",
      "date_format": "DD.MM.YYYY",
      "time_format": "24HR",
      "number_format": "1.234,56",
      "csv_separator": ",",
      "time_display": "HH:MM",
      "week_starts_on": "Monday",
      "timezone": "Berlin",
      "autostop_timers_at": "true",
      "email_on_project_updates": true,
      "email_on_task_updates": true,
      "email_on_subtask_updates": true,
      "email_on_comment_updates": true,
      "weekly_email_report": true,
      "email_on_idle_timer": true,
      "schedule_alerts": null,
      "schedule_alerts_admin": true,
      "snooze_from": null,
      "snooze_to": null,
      "appearance": "dark",
      "appearance_web": "dark",
      "appearance_tray": "dark",
      "appearance_mobile": "auto",
      "file": {
        "name": "f168ad9acbc84ff623bdbc5f72b44c30_1otasbedqg4k08sudel4b35aheasdfasdf.png",
        "file_name": null,
        "path": "",
        "url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/f168ad9acbc84ff623bdbc5f72b44c30_1otasbedqg4k08sudel4b35aheasdfasdf.png",
        "uploaded_by": null,
        "uploaded_at": null,
        "size": null,
        "status": null,
        "object_type": null,
        "object_id": null,
        "object_name": null,
        "mime_type": "png",
        "download_url": null,
        "id": 544804,
        "created_at": "2022-09-29 17:35:26",
        "updated_at": null,
        "json": null
      },
      "timezone_iana": null,
      "id": 134077
    },
    "teams": [
      {
        "account_id": 450410,
        "company": "2. Workspace",
        "status": "REGISTERED",
        "account_status": "SUBSCRIBED",
        "account_token": "5btv4njn67osdb84u8q2eprutk",
        "account_owner": {
          "name": "John",
          "surname": " Doe",
          "avatar_url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/40a7f9ee025e86b2e376a8bd37cb3467_ng03saghs98bnaod2rvcdi6vqb.jpg",
          "token": "CB3UT2IJJSH5CU4A9S0PK7MLK3",
          "third_party_data": null,
          "id": 79634
        },
        "role": "ADMIN",
        "is_archived": false,
        "is_freelance": false,
        "is_selected": false,
        "is_default": false,
        "permissions": {
          "can_edit_time_entries": true,
          "can_edit_projects_and_tasks": true,
          "can_edit_projects": true,
          "can_edit_tasks": true,
          "can_view_time_entries_from_others": true,
          "can_view_others": true
        },
        "plan": 0,
        "active_projects_count": 0,
        "archived_projects_count": 0,
        "active_users_count": 1,
        "archived_users_count": 0,
        "color_index": null,
        "customer_count": 0,
        "service_count": 0,
        "available_storage": null,
        "used_storage": null,
        "metadata": {
          "created_at": "2021-11-23 12:33:20",
          "last_seen_at": "2023-03-08 14:53:51",
          "subscribed_at": null,
          "canceled_at": null,
          "downgraded_at": null,
          "upgraded_subscription_at": null,
          "switched_plan_at": null,
          "deleted_subscription_at": null,
          "deleted_at": null,
          "unpaid_at": null,
          "delinquent_at": null,
          "recovered_at": null,
          "suspended_at": null,
          "gsuite": false,
          "trial_started_at": null,
          "available_storage": 1380000000,
          "used_storage": 0,
          "support_access_until": null
        },
        "integrations": [],
        "settings": {
          "show_message": true,
          "show_step_by_step": true,
          "language": "EN",
          "currency": "USD",
          "currency_name": "United States Dollars - USD",
          "date_format": "MM/DD/YYYY",
          "time_format": "AMPM",
          "number_format": "1,234.56",
          "csv_separator": ",",
          "time_display": "HH:MM",
          "week_starts_on": "Sunday",
          "timezone": "Bucharest",
          "autostop_timers_at": null,
          "email_on_project_updates": null,
          "email_on_task_updates": null,
          "email_on_subtask_updates": null,
          "email_on_comment_updates": null,
          "weekly_email_report": null,
          "email_on_idle_timer": null,
          "schedule_alerts": null,
          "schedule_alerts_admin": null,
          "snooze_from": null,
          "snooze_to": null,
          "appearance": null,
          "appearance_web": null,
          "appearance_tray": null,
          "appearance_mobile": null,
          "file": null,
          "timezone_iana": null,
          "id": 912955
        },
        "logo_url": null,
        "id": 687966,
        "created_at": "2021-11-23 12:33:20",
        "updated_at": null,
        "json": null
      }
    ],
    "permissions": {
      "can_edit_time_entries": true,
      "can_edit_projects_and_tasks": true,
      "can_edit_projects": true,
      "can_edit_tasks": true,
      "can_view_time_entries_from_others": true,
      "can_view_others": true
    },
    "invite_link": null,
    "schedule": {
      "name": "Default Work Schedule",
      "notes": "ho68h7ikbjklorbag2oaucq7e71nlevcoptcqg47d4qn9u7t0ine",
      "status": null,
      "frequency": "WEEKLY",
      "hours": null,
      "from_to": null,
      "mon": 9,
      "tue": 9,
      "wed": 9,
      "thu": 9,
      "fri": 9,
      "sat": null,
      "sun": null,
      "mon_from": "09:00",
      "tue_from": "09:00",
      "wed_from": "09:00",
      "thu_from": "09:00",
      "fri_from": "09:00",
      "sat_from": null,
      "sun_from": null,
      "mon_to": "18:00",
      "tue_to": "18:00",
      "wed_to": "18:00",
      "thu_to": "18:00",
      "fri_to": "18:00",
      "sat_to": null,
      "sun_to": null,
      "notify_users": true,
      "is_default": true,
      "users": null,
      "default_events": null,
      "notifications_frequency": "MONTHLY",
      "show_default_events": true,
      "custom_message": null,
      "notification_day": null,
      "notification_time": null,
      "reminder_frecuency": null,
      "reminder_day": "Friday",
      "reminder_time": null,
      "id": 53287,
      "created_at": "2021-08-26 09:31:05",
      "updated_at": null,
      "json": null
    },
    "preferences": [
      {
        "name": "hours",
        "data": "{\"size\":\"small\",\"stacked\":true,\"snap\":5,\"hide_weekends\":false,\"day_view_calendar\":\"right\"}",
        "id": 135786
      }
    ],
    "apps": [
      {
        "id": 1,
        "key": "timecards",
        "name": "Timecards",
        "notes": "Track clock in and out times for all your employees and create timecards in a breeze.",
        "status": "on",
        "index": 1,
        "release_status": "ga",
        "sub_status": null,
        "prices": null,
        "pricing_model": null,
        "settings": {
          "auto_save_doc": false,
          "auto_submit_days_threshold": 0,
          "auto_submit_timecards": false,
          "remind_users_to_submit": false,
          "monthly_close_date": 31,
          "timecard_range": "monthly",
          "needs_approval": false,
          "allow_breaks": 30,
          "allow_extras": 1,
          "allow_timeoffs": 1,
          "company_details": null,
          "allow_editing_months_threshold": null,
          "allow_editing_days_threshold": 25
        },
        "calendar_settings": null,
        "integrations": null,
        "autotrack_settings": null,
        "account_status": "FREE",
        "type": null
      },
      {
        "id": 2,
        "key": "legacyReports",
        "name": "Legacy Reports",
        "notes": "Get access to old timesheets and custom reports.",
        "status": "off",
        "index": 2,
        "release_status": "ga",
        "sub_status": null,
        "prices": null,
        "pricing_model": null,
        "settings": null,
        "calendar_settings": null,
        "integrations": null,
        "autotrack_settings": null,
        "account_status": "SUBSCRIBED",
        "type": null
      }
    ],
    "employee": null,
    "metadata": {
      "first_seen_at": "2015-09-21 16:00:15",
      "last_seen_at": "2023-03-08 14:56:52",
      "web_last_seen_at": "2023-03-08 14:56:52",
      "mac_last_seen_at": "2023-02-09 17:47:20",
      "chrome_last_seen_at": null,
      "button_last_seen_at": "2020-04-23 17:00:28",
      "tray_last_seen_at": "2022-06-29 15:11:30",
      "hangouts_last_seen_at": null,
      "slack_last_seen_at": null,
      "mobile_last_seen_at": "2020-10-20 17:20:15",
      "teams_last_seent_at": "2020-08-06 16:28:13",
      "email_bounce_reason": null,
      "email_spam": null,
      "preferred_version": "v5"
    },
    "projects_count": 295,
    "custom_fields": null,
    "color_index": 2,
    "tmp_token": null,
    "integrations": null,
    "ctas": null,
    "third_party_data": [],
    "workspaces": null,
    "role_selected": null,
    "id": 79634,
    "created_at": "2015-09-21 16:00:15",
    "updated_at": "2022-12-02 14:13:03",
    "json": "{\"notify_not_tracking\":true,\"notify_when_tracking\":true,\"project_tasks_subtasks_updates\":true,\"detect_idle\":false,\"notify_exceeded_task\":true,\"notify_exceeded_project\":true,\"stop_tracking_when_close\":false,\"disable_all_notifications\":false}"
  }
}
GET
List Users
https://api.trackingtime.co/api/v4/:account_id/users/
Retrieves a list of users using the supported input parameters for filtering results.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
filter
<string>

[ALL | ACTIVE | ARCHIVED]. Default: ACTIVE.

include_billing
<boolean>

Set to true to include billing data such as hourly rate in the response. Default: false.

include_custom_fields
<boolean>

Set to true to include custom fields in the response. Default: false.

PATH VARIABLES
account_id
<integer>

The account id.

Example Request
List Users
curl
curl --location 'https://api.trackingtime.co/api/v4/<integer>/users/' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "id": 381455,
      "name": "John",
      "surname": "Doe",
      "email": "johndoe@domain.com",
      "role": "CO_WORKER",
      "avatar_url": "https://3xo8inm9l8.execute-api.us-east-1.amazonaws.com/image/300x300/0dc5743c1798db6fefff7f90c59c3d47_7k8lnp181mj23vtp3j8enkjkdffas.jpg",
      "is_archived": false,
      "status": "VERIFIED",
      "account_id": 1725,
      "created_at": "2019-02-13 12:10:00",
      "billing": {
        "hourly_rate": 0,
        "hourly_cost": 0
      },
      "invite_link": "https://pro.trackingtime.co/hub/invite?u_token=fjg94hg4j1onnfs11f5r8hp7k6&a_token=6M8O0TKRL5JI48AMMOAEREJQAISDFS",
      "schedule": {
        "id": 53287,
        "name": "Default Work Schedule",
        "status": null,
        "frequency": "WEEKLY",
        "hours": null,
        "from_to": null,
        "mon": 9,
        "tue": 9,
        "wed": 9,
        "thu": 9,
        "fri": 9,
        "sat": null,
        "sun": null,
        "mon_from": "09:00",
        "tue_from": "09:00",
        "wed_from": "09:00",
        "thu_from": null,
        "fri_from": "09:00",
        "sat_from": null,
        "sun_from": null,
        "mon_to": "18:00",
        "tue_to": "18:00",
        "wed_to": "18:00",
        "thu_to": "18:00",
        "fri_to": "18:00",
        "sat_to": null,
        "sun_to": null,
        "notify_users": true,
        "is_default": true,
        "users": null
      },
      "metadata": {
        "first_seen_at": null,
        "last_seen_at": null,
        "web_last_seen_at": null,
        "mac_last_seen_at": null,
        "chrome_last_seen_at": null,
        "button_last_seen_at": null,
        "tray_last_seen_at": null,
        "hangouts_last_seen_at": null,
        "slack_last_seen_at": null,
        "mobile_last_seen_at": null,
        "teams_last_seent_at": null,
        "email_bounce_reason": null,
        "email_spam": null,
        "preferred_version": null
      },
      "worked_hours_this_month": null,
      "worked_hours_this_week": null,
      "worked_hours_today": null,
      "loc_worked_hours_this_month": null,
      "loc_worked_hours_this_week": null,
      "loc_worked_hours_today": null
    },
    {
      "id": 542414,
      "name": "Alice",
      "surname": "Doe",
      "email": "alicedoe@domain.com",
      "role": "ADMIN",
      "avatar_url": "https://www.gravatar.com/avatar/53abcd46829e7e4809d0c3bdfdc316f5SDFS.jpg?d=null",
      "is_archived": false,
      "status": "VERIFIED",
      "account_id": 1725,
      "created_at": "2022-04-08 11:10:00",
      "billing": {
        "hourly_rate": 0,
        "hourly_cost": 0
      },
      "invite_link": "https://dev.trackingtime.io/hub/invite?u_token=vad2vha2vmomg1pdsplao7uhfk&a_token=6M8O0TKRL5JI48AMMOAEREJQAISDFDSF",
      "schedule": {
        "id": 53287,
        "name": "Default Work Schedule",
        "status": null,
        "frequency": "WEEKLY",
        "hours": null,
        "from_to": null,
        "mon": 9,
        "tue": 9,
        "wed": 9,
        "thu": 9,
        "fri": 9,
        "sat": null,
        "sun": null,
        "mon_from": "09:00",
        "tue_from": "09:00",
        "wed_from": "09:00",
        "thu_from": null,
        "fri_from": "09:00",
        "sat_from": null,
        "sun_from": null,
        "mon_to": "18:00",
        "tue_to": "18:00",
        "wed_to": "18:00",
        "thu_to": "18:00",
        "fri_to": "18:00",
        "sat_to": null,
        "sun_to": null,
        "notify_users": true,
        "is_default": true,
        "users": null
      },
      "metadata": {
        "first_seen_at": null,
        "last_seen_at": null,
        "web_last_seen_at": null,
        "mac_last_seen_at": null,
        "chrome_last_seen_at": null,
        "button_last_seen_at": null,
        "tray_last_seen_at": null,
        "hangouts_last_seen_at": null,
        "slack_last_seen_at": null,
        "mobile_last_seen_at": null,
        "teams_last_seent_at": null,
        "email_bounce_reason": null,
        "email_spam": null,
        "preferred_version": null
      },
      "worked_hours_this_month": null,
      "worked_hours_this_week": null,
      "worked_hours_today": null,
      "loc_worked_hours_this_month": null,
      "loc_worked_hours_this_week": null,
      "loc_worked_hours_today": null
    }
  ]
}
GET
List User Tasks
https://api.trackingtime.co/api/v4/:account_id/users/:user_id/assigned_tasks
Retrieves a list of user tasks using the supported input parameters for filtering results.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
filter
<string>

[ALL | ACTIVE | ARCHIVED | TRACKING]. Use the TRACKING filter to retrieve all account tasks currently tracking. Default = ACTIVE.

page
<integer>

Use for pagination. Default: 0.

page_size
<integer>

Use for pagination. Default: 100.

sort_by
<string>

Used to sort tasks in a specific order. Supported values: [id | name | priority | project | customer | due_date | closed_date]. Default: name.

order
<string>

[ASC | DESC]. Default: ASC.

include_billing
<boolean>

Set to 'true' to include billing data in the response. Default = false.

PATH VARIABLES
account_id
<integer>

The account id.

user_id
<integer>

The user id.

Example Request
List User Tasks
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/<integer>/users/<integer>/assigned_tasks' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "account_id": 1,
      "id": 9465533,
      "name": "",
      "type": "PERSONAL",
      "color": null,
      "index": 0,
      "estimated_time": 0,
      "accumulated_time": 0,
      "accumulated_time_display": "00:00:00",
      "worked_hours": 0,
      "loc_worked_hours": "00:00",
      "due_date": null,
      "is_archived": false,
      "visibility": null,
      "project_id": 12669262123,
      "project": "Sample Project",
      "project_following": false,
      "project_is_favorite": false,
      "customer": null,
      "customer_id": null,
      "service": null,
      "service_id": null,
      "list_id": null,
      "list_name": null,
      "list_position": 0,
      "tracking": false,
      "users": [
        {
          "id": 542588,
          "name": "John",
          "surname": "Doe",
          "avatar_url": "https://lh3.googleusercontent.com/a/AEdFTp7KxvqBPlJV1vPt3VXesOV2biHZQXc3-HlRlzvB=s96asdf-c",
          "status": "ACTIVE",
          "shared_on": "2023-02-08 16:34:39",
          "event": null,
          "estimated_time": null,
          "accumulated_time": 0
        }
      ],
      "created_at": null,
      "third_party_data": null,
      "third_party_project": null
    },
    {
      "account_id": 1,
      "id": 9465590232,
      "name": "",
      "type": "PERSONAL",
      "color": null,
      "index": 0,
      "estimated_time": 0,
      "accumulated_time": 0,
      "accumulated_time_display": "00:00:00",
      "worked_hours": 0,
      "loc_worked_hours": "00:00",
      "due_date": null,
      "is_archived": false,
      "visibility": null,
      "project_id": null,
      "project": null,
      "project_following": false,
      "project_is_favorite": false,
      "customer": null,
      "customer_id": null,
      "service": null,
      "service_id": null,
      "list_id": null,
      "list_name": null,
      "list_position": 0,
      "tracking": false,
      "users": [
        {
          "id": 542588212,
          "name": "John",
          "surname": "Doe",
          "avatar_url": "https://lh3.googleusercontent.com/a/AEdFTp7KxvqBPlJV1vPt3VXesOV2biHZQXc3-HlRlzvB=s96asdf-c",
          "status": "ACTIVE",
          "shared_on": "2023-03-03 15:36:52",
          "event": null,
          "estimated_time": null,
          "accumulated_time": 0
        }
      ],
      "created_at": null,
      "third_party_data": null,
      "third_party_project": null
    }
  ]
}
GET
List User Projects
https://api.trackingtime.co/api/v4/:account_id/users/:user_id/projects
Retrieves a list of user projects using the supported input parameters for filtering results.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PATH VARIABLES
account_id
<integer>

The account id.

user_id
<integer>

The user id.

Example Request
List User Projects
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/<integer>/users/<integer>/projects' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "projects": [
      {
        "start_date": null,
        "loc_start_date": null,
        "end_date": null,
        "loc_end_date": null,
        "delivery_date": null,
        "loc_delivery_date": null,
        "estimated_time": 20,
        "loc_estimated_time": null,
        "accumulated_time": 0,
        "is_archived": false,
        "following": false,
        "notes": "",
        "name": "Sample Project",
        "color": null,
        "tasks": null,
        "task_lists": null,
        "customer": null,
        "service": null,
        "billing": null,
        "is_public": null,
        "default_view": null,
        "worked_hours": 0,
        "loc_worked_hours": "00:00",
        "worked_hours_today": null,
        "worked_hours_this_week": null,
        "worked_hours_this_month": null,
        "loc_worked_hours_today": null,
        "loc_worked_hours_this_week": null,
        "loc_worked_hours_this_month": null,
        "active_tasks": null,
        "archived_tasks": null,
        "is_template": false,
        "users_hours": null,
        "coworkers": null,
        "files": null,
        "account_index": null,
        "user_preferences": null,
        "doc_id": null,
        "third_party_data": null,
        "user_index": null,
        "id": 154208412,
        "created_at": "2022-09-13 09:19:52",
        "updated_at": null,
        "json": null
      }
    ]
  }
}
GET
List User Trackables
https://api.trackingtime.co/api/v4/:account_id/users/:user_id/trackables
This endpoint returns projects and tasks assigned to any given user. Trackables reprent projects and tasks, which a user can track time against.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PARAMS
project_id
<integer>

The id of an existing projects to filter by. You should avoid setting include_tasks to false when using this filter. Use project_id = 0 to retrieve only tasks that are currently not linked to any project.

only_favorites
<boolean>

Set to true to include only projects marked as favorite by the requesting user.

include_tasks
<boolean>

Set to false to return only projects. Default: true.

PATH VARIABLES
account_id
<integer>

The account id.

user_id
<integer>

The user id.

Example Request
List User Trackables
curl
curl --location 'https://api.trackingtime.co/api/v4/users/<integer>/trackables' \
--header 'Authorization;'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "projects": [
      {
        "id": 1261237058,
        "name": "Sample Project",
        "color": null,
        "is_public": false,
        "is_favorite": false,
        "customer": {
          "name": "Sample Customer",
          "id": 372219
        },
        "tasks": [
          {
            "id": 9465590123,
            "name": "",
            "type": "PERSONAL",
            "estimated_time": null,
            "accumulated_time": 0,
            "due_date": null,
            "users": [],
            "task_permalink": null
          },
          {
            "id": 94612315602,
            "name": "",
            "type": "PERSONAL",
            "estimated_time": null,
            "accumulated_time": 0,
            "due_date": null,
            "users": [],
            "task_permalink": null
          }
        ]
      },
      {
        "id": 0,
        "name": "No Project",
        "color": null,
        "is_public": null,
        "is_favorite": null,
        "customer": null,
        "tasks": []
      }
    ]
  }
}
GET
List Enum Options
https://api.trackingtime.co/api/v4/:account_id/custom_fields/:custom_field_id/enum_options
Retrieves a list of enum options using the supported input parameters for filtering results.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
PARAMS
enabled
<boolean>

Filter by current status. Default: true.

PATH VARIABLES
account_id
<integer>

The account id.

custom_field_id
<integer>

The custom field id.

Example Request
List Enum Options
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/1725/custom_fields/3404/enum_options?enabled=false'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": [
    {
      "id": 1271,
      "name": "Another name",
      "color": null,
      "index": 2,
      "enabled": false
    }
  ]
}
POST
Update an Enum Option
https://api.trackingtime.co/api/v4/:account_id/enum_options/:enum_option_id/update
Updates an existing Enum Option identified by a unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
PARAMS
name
<string>

Human-friendly identifier.

PATH VARIABLES
account_id
<integer>

The account id.

enum_option_id
Example Request
Update Enum Option
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/1725/enum_options/1271/update?name=Another%20name'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "id": 1271,
    "name": "Another name",
    "color": null,
    "index": 2,
    "enabled": false
  }
}
GET
Delete an Enum Option
https://api.trackingtime.co/api/v4/:account_id/enum_options/:enum_option_id/delete
⛔️ WARNING: This action cannot be undone.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
HEADERS
Authorization
PATH VARIABLES
account_id
<integer>

The acount id.

enum_option_id
<integer>

The enum option id.

Example Request
Delete Enum Option
curl
curl --location 'https://api.trackingtime.co/api/v4/1725/enum_options/1271/delete'
200 OK
Example Response
Body
Headers (13)
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {}
}
POST
Add a Custom Field
https://api.trackingtime.co/api/v4/:account_id/custom_fields/add
Creates a new custom field.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
PARAMS
name
<string>

Required. Human-friendly identifier. Unique for each filter_object_class.

color
<string>

Optional. The color in which the custom field will be displayed.

notes
<string>

Optional. Describe how the custom field should be use for.

value_type
<string>

Required. The type of the value the custom field supports. Supported values: [boolean | currency | date | enum | number | text | checkbox].

filter_object_class
<string>

Required. Specifies the object class the custom field should be linked to. Supported values: [event | task | project | user | customer].

cf_index
<integer>

Optional. Specifies the order in which the custom field will be displayed.

PATH VARIABLES
account_id
<integer>

The account id.

Example Request
Add Custom Field
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/:account_id/custom_fields/add?name=Custom%20Field%20Name&value_type=text&filter_object_class=project'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "id": 6271,
    "name": "Custom Field Name",
    "value": null,
    "color": null,
    "type": null,
    "notes": null,
    "value_type": "text",
    "value_object_class": null,
    "filter_object_class": "project",
    "is_object_default": false,
    "status": "ACTIVE",
    "enum_options": null,
    "cf_index": null,
    "slug": null
  }
}
POST
Update a Custom Field
https://api.trackingtime.co/api/v4/:account_id/custom_fields/:custom_field_id/update
Updates a single custom field identified by its unique id.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
PARAMS
name
<string>

Required. Human-friendly identifier. Unique for each filter_object_class.

color
<string>

Optional. The color in which the custom field will be displayed.

notes
<string>

Optional. Describe how the custom field should be use for.

value_type
<string>

Required. The type of the value the custom field supports. Supported values: [boolean | currency | date | enum | number | text | checkbox].

filter_object_class
<string>

Required. Specifies the object class the custom field should be linked to. Supported values: [event | task | project | user | customer].

cf_index
<integer>

Optional. Specifies the order in which the custom field will be displayed.

PATH VARIABLES
account_id
<integer>

The account id.

custom_field_id
<integer>

The custom field id.

Example Request
Update Custom Field
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/:account_id/custom_fields/:custom_field_id/update?name=Name%20Edited'
200 OK
Example Response
Body
Headers (13)
View More
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {
    "id": 6271,
    "name": "Name Edited",
    "value": null,
    "color": null,
    "type": null,
    "notes": null,
    "value_type": "project",
    "value_object_class": null,
    "filter_object_class": null,
    "is_object_default": false,
    "status": "ACTIVE",
    "enum_options": null,
    "cf_index": null,
    "slug": null
  }
}
GET
Delete a Custom Field
https://api.trackingtime.co/api/v4/:account_id/custom_fields/:custom_field_id/delete
⛔️ WARNING: This action cannot be undone.

AUTHORIZATION
Basic Auth
This request is using Basic Auth from collectionTrackingTime API
PATH VARIABLES
account_id
<integer>

The account id.

custom_field_id
<integer>

The custom field id.

Example Request
Delete Custom Field
View More
curl
curl --location 'https://api.trackingtime.co/api/v4/:account_id/custom_fields/:custom_field_id/delete'
200 OK
Example Response
Body
Headers (13)
json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {}
}
Release Notes
11 August 2025
API v4.95.62
Feature: Assign events to multiple tasks

Feature: Return list_id when update a task

05 August 2025
API v4.95.61
Feature: Allow setting calendar default task to null.

Feature: Validate trial started at

Feature: Change repeat events to 5 years

30 July 2025
API v4.95.60
Fix: Delete enum custom field
24 July 2025
API v4.95.59
Feature: Refactor to calculate auto sum
22 July 2025
API v4.95.58
Feature: Free and starter must be admin
21 July 2025
API v4.95.57
Feature: shorten sync calendar data range creating three months window
03 July 2025
API v4.95.56
Feature: AutoSum project.estimated_time when sum_estimates is true

Feature: List events min: holiday by user

Fix: Added new catch for Hibernate StaleStateException to allow restarting the import of calendar events.

Feature: List events min: holiday by user

Fix: CSV import: events and tasks now assigned by customer name

03 July 2025
API v4.95.55
Feature: list evets Min: new parameter, is_paid_timeoff
26 June 2025
API v4.95.54
Feature: New endpoint Dashboard
10 June 2025
API v4.95.53
Feature: Adding filter for google calendar holiday events

Feature: Refactor when to and from are null when including timeoff in events min

10 June 2025
API v4.95.52
Feature: Add deductible_formatted and proportional_deductible_formatteded parameters in events min

Fix: Duplicate timeoff in events min

04 June 2025
API v4.95.51
Fix: Admin can edit rejected reason
27 May 2025
API v4.95.50
Fix: Remove limit when get custom field.
27 May 2025
API v4.95.49
Fix: weekly report.

Feature: Search 2.0

26 May 2025
API v4.95.48
Fix: Update limit custom field.
21 May 2025
API v4.95.47
Feature: Tracking policies : prevenir edición de events x working days in the past
20 May 2025
API v4.95.46
Fix: Fix double email
13 May 2025
API v4.95.45
Fix: Lidl holiday, default events
12 May 2025
API v4.95.44
Fix: Update Microsoft client secrets.
09 May 2025
API v4.95.43
Feature: Add holiday event to lidl handle events.
07 May 2025
API v4.95.42
Fix: Timeoff adjustment duplicated

Fix: List event wrong duration

Feature: Share task, cant create new tasks when project is archived

Fix: Tasklist fav websockets

28 April 2025
API v4.95.41
Feature: Added cat and uat in list event min.
21 April 2025
API v4.95.40
Fix: Timecards for timeoffs
14 April 2025
API v4.95.39
Fix: Check satus OK in google_calendar_external_id before to send email
10 April 2025
API v4.95.38
Fix: Update batshsize in calendar job 3 and 4
08 April 2025
API v4.95.37
Fix: Timeoff all day false in list event min
04 April 2025
API v4.95.36
Feature: Added new paramaters to list calendars
28 March 2025
API v4.95.35
Fixed: Removed Sentry logs from calendar job step four
27 March 2025
API v4.95.34
Fixed: Sentry logs to step four calendar

Fixed: Update event in batch

26 March 2025
API v4.95.33
Feature: Sentry logs to step four calendar
25 March 2025
API v4.95.32
Fix: Timecard overlaping

Feature: Pricing updates

Feature: List doc versions

Feature: Added start and end source to list tomeoffs

19 March 2025
API v4.95.31
Fix: Is billable
18 March 2025
API v4.95.30
Feature: set Blocked status for update event in batch
12 March 2025
API v4.95.29
Feature: Mandatory custom field
12 March 2025
API v4.95.28
Feature: Fixed events min get timeoffs when also has time entry
07 March 2025
API v4.95.27
Feature: Weekly report with timeoffs and holidays
05 March 2025
API v4.95.26
Feature: Recalculate Time off by Schedule
19 February 2025
API v4.95.25
Feature: Improved sort by project for list assigned task

Feature: Time off

20 February 2025
API v4.95.24
Fixed: Removed change start when periodo is all_time getting doc
19 February 2025
API v4.95.23
Feature: Coworker if Can Edit tasks = false and Can edit projects = false enabled smart add
13 February 2025
API v4.95.22
Feature: Improvement of sort project in list assigned tasks
10 February 2025
API v4.95.21
Fix: Fixed inbound sengrid for outlook
06 February 2025
API v4.95.20
Feature: Added color to event min
05 February 2025
API v4.95.19
Fix: Time off

Fix: Fix sendgrid inbound parse when email come from apple

04 February 2025
API v4.95.18
Fix: Time off
04 February 2025
API v4.95.17
Fix: Sendgrid Inbound parse
30 January 2025
API v4.95.16
Fix: Get Doc last_week / past_week

Fix: Set TimeoffStatus

27 January 2025
API v4.95.15
Fix: Add Log TimeZone null

Feature: Add Time Off App Settings

21 January 2025
API v4.95.14
Feature: Add Log TimeZone null

Fixed: User Log TimeOff

20 January 2025
API v4.95.12
Fixed: Query Assigned Task
15 January 2025
API v4.95.10
Fixed: Time Off Fixes
14 January 2025
API v4.95.09
Fixed: Prevent deletion of time entry

Feature: New job work plan exclusive

Feature: Endpoint timecard count

Fixed: Bamboo import duplication

02 January 2025
API v4.95.08
Fixed: User task filter by date

Feature: New parameter is deductible for list event min

Feature: Plan Freelancer

02 January 2025
API v4.95.07
Feature: Do not delete schedule when archive user

Feature: Project link

Feature: Time card default

Feature: Rate and cost

30 December 2024
API v4.95.06
Fix: Time Off Fixes
20 December 2024
API v4.95.05
Fix: Time Off Fixes
19 December 2024
API v4.95.04
Feature: Migrate Timeoff to Adjustment Backoffice
16 December 2024
API v4.95.03
Feature: Added custom field to import event, task and user

Fix: Fixed when set up an default time off type

16 December 2024
API v4.95.02
Fix: Fixed weekly report

Fix: Fixed update doc, update only present params

12 December 2024
API v4.95.01
Fix: Fixed grammmar sql exception in getTimeoffs

Feature: In endpoint user, added created_at when user is coworker

Feature: New Endpoint filtered events

Feature: Added customfilds into websocket events

Fix: Fixed missing index in tasks share

11 December 2024
API v4.95.00
Feature: Time off V2
27 November 2024
API v4.94.60
Fix: Fixed Time off timecard

Fix: Fixed filters in private list docs events

14 Noviembre 2024
API v4.94.59
Feature: Add Column Holiday in Timecard Report

Fix: Fixed policy when trying to check PMs can edit Event

5 Noviembre 2024
API v4.94.58
Fix: Fix default holidays hours / seconds
30 October 2024
API v4.94.57
Feature: Add Log BaseIntegration
28 October 2024
API v4.94.56
Feature: Add Seconds in Timecard

Fix: Money Mode Queue

Fix: Fixed stripe

07 October 2024
API v4.94.55
Fix: Change Scope Microsoft
01 October 2024
API v4.94.54
Fix: TimeCardGeneratorJob fixed adjustment date
01 October 2024
API v4.94.52
Fix: Changed message into Microsoft Domain Connector

Fix: Live report

27 September 2024
API v4.94.51
Fix: Fixed time card generator job
3 September 2024
API v4.94.50
Fix: Fixed stripe

Fix: Fixed delete task restore

Feature: Add Error Site html

6 August 2024
API v4.94.49
Feature: Fixed error into timeoff
6 August 2024
API v4.94.48
Feature: Lidl show timeoff over work
29 July 2024
API v4.94.47
Fix: Fix Custom Field Value in Flat Action
17 July 2024
API v4.94.46
Fix: Fix Bug in Recovery Task
03 July 2024
API v4.94.45
Fix: Change Max limit in custom field query
24 June 2024
API v4.94.44
Fix: Fix state deleted in timeoff Bamboo
18 June 2024
API v4.94.43
Fix: Fix separator in the redirect URL
10 June 2024
API v4.94.42
Feature: Prevent generated zero timecards
29 May 2024
API v4.94.41
Feature: Delete Old events in Bamboo

Feature: Add New Log Timecard

Fix: Added LastName in BI

14 May 2024
API v4.94.40
Feature: ChatGPT improvements

Feature: Set team as default when cancel account

Feature: Added validation in add Doc

6 May 2024
API v4.94.39
Fix: Holiday in timecard.
23 April 2024
API v4.94.38
Feature: Add login with OAuth.
23 April 2024
API v4.94.37
Fix: Fixed NPE in getDoc API.
22 April 2024
API v4.94.36
Fix: Changed start and end when getDoc have a period.
17 April 2024
API v4.94.35
Fix: Fix can edit range date in update doc
19 March 2024
API v4.94.34
Feature: Increasing the expiration time of the S3 files to 30 days

Feature: Added limit to 100 in delete event in batch

Feature: Added custom fields batch add user

Fix: Added task lists in private and public get project doc

07 March 2024
API v4.94.33
FIX: TimeCardsDelegate getStringWithOutLimit entries

FIX: Duplicate event list regarding to project_membership

06 March 2024
API v4.94.32
FEATURE: Added parameters index, hourly_rate and is_billble in update task in batch

FIX: Set addSessionAccountToStripeQueue

FIX: Set hours in BambooConnector

FIX: CF when duplicate project

FIX: Stop Auto charging open invoices in stripe webhooks

28 February 2024
API v4.94.31
FEATURE: Time Cards Supervisor
26 February 2024
API v4.94.30
FEATURE: Added project custom field when duplicate project.

FEATURE: Added task custom field when duplicate project.

FEATURE: Added to event keys from filter and groups.

FIX: Grammar SQL Exception when get task assignments.

3 Abril 2024
API v4.94.30
FEATURE: Add Log Multiple Request.

FIX: Change duration share link aws.

1 February 2024
API v4.94.29
FEATURE: Add change app SSO.

FIX: Remove validation .ru by subscribed.

20 February 2024
API v4.94.28
FEATURE: Add log Add Account To Stripe Queue.

FIX: Add Validate new user in ImportTasksAction.

08 February 2024
API v4.94.27
Fix: Changed getString to getStringWithoutLimit

New: Added scheduled report to run all jobs

08 February 2024
API v4.94.26
Fix: Ghost task of non-billable project

Fix: Storage calculation.

07 February 2024
API v4.94.25
Fix: Close user.

Fix: Time Off.

Fix: Half Past Time Off in BambooConnector.

New: Changed Parameters for scheduled report.

Fix: Assigned tasks, grammar sql error.

06 February 2024
API v4.94.24
NEW: Added scheduled live reports.
30 January 2024
API v4.94.23
FEATURE: Change number MAX_ACCOUNT_ACTIVE_USERS_ON_TRIAL.
30 January 2024
API v4.94.22
NEW: Block Email .ru.
22 January 2024
API v4.94.21
IMPROVEMENT: Set false in import user BambooHr Sync.

FIX: Fix Freelancer Update.

NEW: Stripe Auditor Subscription Job.

NEW: Stripe Auditor Delinquents Job.

NEW: Support access login with SSO.

9 January 2024
API v4.94.20
FIX: Fix transaction in Delete Event Notify.
8 January 2024
API v4.94.19
IMPROVEMENT: A filter was added for the 'type' column, excluding the 'SYSTEM' values.

IMPROVEMENT: Send Email Timeoff.

NEW: Added safe deleted event endpoint.

20 December 2023
API v4.94.18
NEW: Implement google SSO.
20 December 2023
API v4.94.17
IMPROVEMENT: Remove Upper in query.
20 December 2023
API v4.94.16
FIX: Bug fix in task deletion without projects.

IMPROVEMENT: Addition of webhook sending in a separate thread in update project.

12 December 2023
API v4.94.15
IMPROVEMENT: Added support agent access when SSO is enabled.

NEW: Added soft task delete.

IMPROVEMENT: Added filter to avoid syncing canceled events of Microsoft calendars.

1 December 2023
API v4.94.14
FIX: Added support for plan type "flexible" in backoffice update account endpoint.

FIX: Fixed set user en log delete.

26 November 2023
API v4.94.13
FIX: Fixed issue in backoffice endpoint to switch annual customers to new pricing.
25 November 2023
API v4.94.12
FIX: Fixed billed and is_billable issue for PMs.

FIX: Fixed update domain type to always set google or microsoft.

24 November 2023
API v4.94.11
FIX: Added advanced logs in Stripe cronjob for better debugging.

FIX: Fixed no session error in delete tasks in batch endpoint.

FIX: Fixed min seats calculation NPE in Stripe's subscription updated webhook.

23 November 2023
API v4.94.10
FIX: Turned off WebhookDataSyncerJob concurrent executions.

IMPROVEMENT: Avoid updating account subscription parameters twice when switching to an annual plan.

NEW: Add default error redirects in subscription portal and checkout endpoints.

FIX: Fixed webhook queue project id when recalculating projects' cost and rates.

FIX: Fixed configuration error in switch to annual endpoint.

NEW: Added proxy endpoint to download s3 files and return better error messages if any.

IMPROVEMENT: Improved webhooks queue validation filters to avoid adding unnecessary projects to the queue.

IMPROVEMENT: Added account status validation when checking whether the money mode add-on is enabled or not.

FIX: Fixed is_billable method to return true when just one parameter (hourly rate, hourly cost or fixed rate) is true.

IMPROVEMENT: Refactored webhook data processor to use multiple transactions.

16 November 2023
API v4.94.9
FIX: Fixed issue in list projects for coworkers.
15 November 2023
API v4.94.8
IMPROVEMENT: Improved german copy.

FIX: Fixed NPE when user id is null in update account status endpoint.

NEW: Implemented scripts to further automate deployments.

14 November 2023
API v4.94.7
NEW: Added new 2023 pricing plans and improved customer portal experience.

IMPROVEMENT: Improved error message for account not found in SSO.

IMPROVEMENT: Updated login endpoint to return redirect message when SSO enabled.

FIX: Fixed custom field types validation.

FIX: Fixed null pointer exception in calendar step four when event is null.

FIX: Fixed json error in list flat events endpoint.

FIX: Fixed returned slug parameter. Added to teams node.

NEW: Added process payment success webhook in Stripe.

FIX: Avoid duplicating docs when a project is duplicated.

IMPROVEMENT: Added automatic charge for open invoices in payment succeeded Stripe webhook.

IMPROVEMENT: Updated list paginated tasks endpoint to set the default page size to 500 and max custom field size to 10,000.

FIX: Fixed NPE and removed call to calculateAccumulatedProject method in add / update task / event endpoints.

7 November 2023
API v4.94.6
FIX: Fixed task comments and task logs timezone issues.

IMPROVEMENT: Added is_billable and is_billed to list events endpoint even when the add-on Cost and Rates is turned off.

NEW: Added list and email types for custom fields.

FIX: Fixed user role validation in add and update customer endpoints.

NEW: Added phone, percentage, url and rating types for custom fields.

NEW: Added list flat events endpoint to retrieve event and custom field data in one array.

FIX: Fixed error message in login endpoint when the user correctly authenticated, but currently archived.

FIX: Fixed validation issue in update access policies endpoint.

FIX: Removed account creation date when changing status of cost and rates add-on.

IMPROVEMENT: Added user log in join account via domain capture.

IMPROVEMENT: Added enabled parameter for custom fields to allow admins to switch custom fields on and off.

FIX: Fixed upper case issue in the account slug in the signin with SSO endpoint.

NEW: Added new plan validations for 2023 pricing plans.

IMPROVEMENT: Added auto-redirect to SSO homepage when a user tries to login using email and password when SSO is enabled in the account.

IMPROVEMENT: Added sort assigned tasks by due date and priority.

27 October 2023
API v4.94.5
FIX: Fixed task object class for calendar sync custom field.

FIX: Fixed name and surname null in Sign-in with Microsoft.

FIX: Fixed regular expression to improve domain parsing.

FIX: Fixed a wrong property in verify Google domain endpoint.

IMPROVEMENT: Updated duplicate task endpoint to copy all task custom fields.

25 October 2023
API v4.94.3
FIX: Fixed an issue with user rates validity periods.

IMPROVEMENT: Started marking events synced with a third party calendar as calendar events.

IMPROVEMENT: Started marking tasks synced with a third party calendar as calendar sync tasks.

FIX: Fixed saml properties used for SSO with Azure AD.

25 October 2023
API v4.94.2
FIX: Fixed SSO redirect URI issue.

FIX: Fixed an issue in stop tracking task endpoint when auto-stopping the task due to incorrect dates.

FIX: Updated sync calendar endpoint to block pre-defined users from turning the synchronization on.

FIX: Fixed an issue in event endpoints where the incorrect object id would be saved in the webhooks queue.

IMPROVEMENT: Improved calendar revoked access notification.

IMPROVEMENT: Stopped adding 'add' or 'delete' type items to the 'third_party_cache' table from the Trello job.

FIX: Fixed an issue in domain capture to avoid registering the same domain twice.

IMPROVEMENT: Added advanced logs in Trello cronjobs.

IMPROVEMENT: Added stop tracking task threshold property.

FIX: Fixed domain entity mapping.

25 October 2023
API v4.94.1
NEW: Updated calendar cronjobs to mark tasks as calendar tasks.

NEW: Updated calendar cronjobs to mark imported events as calendar events.

FIX: Fixed SSO production properties.

24 October 2023
API v4.94.0
FIX: Fixed policy issues when turning on calendars if the account has default events set up.

IMPROVEMENT: Improved calendar revoked access email notification.

IMPROVEMENT: Refactored Trello cronjobs to avoid adding "add" or "delete" type items to the third party cache.

IMPROVEMENT: Improved Trello cronjobs to speed up sync process.

IMPROVEMENT: Added request logs to Trello requests.

FIX: Fixed timezone issue in sync tracking task endpoint.

NEW: Added domain management feature.

NEW: Added SSO with Azure AD feature.

10 October 2023
API v4.93.18
FIX: Fixed recover password issue when user has not yet accepted their invite.

IMPROVEMENT: Added support for saving calendar refresh tokens to avoid revoked tokens issues.

IMPROVEMENT: Improved error handling and logging in Trello cronjobs.

IMPROVEMENT: Increased report file's presigned expiration time to 24h.

04 October 2023
API v4.93.17
FIX: Refactored accept invite endpoint to improve error handling and performance monitoring.

FIX: Deleted deprecated mappings in metadata objects.

FIX: Fixed name and surname in microsoft signin callback endpoint.

NEW: Added safe deleted tasks endpoint.

FIX: Removed free_pending filter in Trello cronjobs to improve performance.

26 September 2023
API v4.93.16
IMPROVEMENT: Replaced default CSV separator in export account data. Now using simple comma.

NEW: Added close and open customer endpoints and updated list customers endpoint to filter results by status.

FIX: Fixed session close issue in calendar cronjobs.

FIX: Fixed NPE in microsoft connector when username or last name are null.

11 September 2023
API v4.93.15
IMPROVEMENT: Added close and open task to Zapier webhooks when a task is updated.

IMPROVEMENT: Added custom fields support to list tasks endpoint.

IMPROVEMENT: Updated delete task endpoint to block deleting caledar sync tasks.

FIX: Turned off database logging.

FIX: Fixed NPE when calendar is null.

NEW: Marked calendar events as deleted when a task is deleted.

FIX: Fixed account status filter to process calendar sync.

FIX: Fixed missing calendar query parameters.

05 September 2023
API v4.93.14
FIX: Removed debug logs from default logging.

IMPROVEMENT: Added include_custom_fields to list paginated tasks endpoint.

30 August 2023
API v4.93.13
FIX: Fixed NPE when adding user log in add event endpoint.

IMPROVEMENT: Filtered out-of-office events from synced calendars.

28 August 2023
API v4.93.12
FIX: Fixed an issue sending Stripe job error count to New Relic.

IMPROVEMENT: Added exponential backoff retry in hasOccurrenceWithinRange in calendar cronjobs.

IMPROVEMENT: Added API request log errors in New Relic.

IMPROVEMENT: Added Workplan Weekly cronjob to start all schedulers endpoint.

IMPROVEMENT: Added processed records logs in Webhooker Data Processor.

IMPROVEMENT: Improved calendar error messages.

FIX: Updated calendar cronjobs to prevent dropping calendars after 5xx errors.

FIX: Excluded Hibernate warning logs in New Relic.

IMPROVEMENT: Added calendar events processed count logs.

FIX: Fixed NPE when calendar not found.

FIX: Updated calendar cronjobs to prevent dropping MS calendar after client-secret expires.

FIX: Fixed issue in search projects where deleted projects would be included in results.

07 August 2023
API v4.93.11
FIX: Updated add event endpoint to avoid policy checking when adding events from third party services.

FIX: Fixed an issue in the add event endpoint where custom fields would fail when third party service provided.

FIX: Fixed autostop timezone issue in sync task endpoint.

IMPROVEMENT: Updated list events query to include events with duration zero.

IMPROVEMENT: Added cronjob logs to New Relic.

IMPROVEMENT: Added auto_advance true when creating one-off invoices in Stripe.

FIX: Fixed selection query in weekly report cronjob.

IMPROVEMENT: Improved connection handling in Trello cronjobs.

31 July 2023
API v4.93.10
IMPROVEMENT: Updated delete task endpoint to block deleting calendar sync tasks.

IMPROVEMENT: Updated Trello Connector to automatically turn the app off when a revoked access exceptions occurs.

IMPROVEMENT: Deprecated X-Ray filters from the web app.

IMPROVEMENT: Added automatic retries in Microsoft Calendar connector when a request timeout exception occurs.

25 July 2023
API v4.93.9
FIX: Fixed limits in select set custom fields in list customers endpoint.

FIX: Fixed an issue when marking Microsoft calendar events as deleted.

IMPROVEMENT: Updated search endpoints to allow keys of one character.

FIX: Fixed NPE in localization of Sentry events.

FIX: Fixed an issue in update event endpoint where users could not update the task of other users' events.

24 July 2023
API v4.93.8
IMPROVEMENT: Refactored duplicated code in Microsoft Calendar connector.

FIX: Several fixes and improvements for Calendar connectors.

IMPROVEMENT: Added per-user quota to Google Calendar connector to avoid hitting API rate limits.

FIX: Fixed several Sentry issues.

IMPROVEMENT: Improved database query for the Timecop cronjob.

IMPROVEMENT: Improved database query for the Reminder cronjob.

IMPROVEMENT: Improved database query for the Webhooks Cleaner cronjob.

IMPROVEMENT: Added exponential backoff to avoid timeouts in Microsoft Calendar connector.

FIX: Fixed filters in search query.

FIX: Fixed NPE when verifying sync in progress in Calendar connectors.

FIX: Fixed exception in grant edit access endpoint.

IMPROVEMENT: Added default timeouts to list user logs and list projects endpoint to avoid database performance issues.

IMPROVEMENT: Removed update metadata operations in login endpoint to avoid lock issues.

IMPROVEMENT: Improved error handling in sync task endpoint.

FIX: Fixed task tracking issue in update event endpoint where the currently active task would get stopped when editing an event for the same task.

IMPROVEMENT: Added websocket messages in add task comment endpoint.

IMPROVEMENT: Refactored list assigned tasks endpoint to include ghost tasks.

FIX: Fixed an issue in add task list endpoint where the project would be invalid.

FIX: Fixed NPE in resend verification email.

FIX: Removed all control characters from JSON notifications to avoid encoding issues.

FIX: Fixed NPE when integration not found in Microsoft Connector.

IMPROVEMENT: Improved Google Calendar connector to automatically retry requests after socket timeout exceptions.

IMPROVEMENT: Improved retry mechanism if rate limit exceeded in Google Calendar connector.

IMPROVEMENT: Added message keys to Sentry events to improve event grouping.

FIX: Fixed an issue where the intercom update would get triggered on demand.

FIX: Fixed NPE in list audit logs endpoint.

FIX: Fixed NPE in Intercom Data Processor cronjob.

17 July 2023
API v4.93.7
FIX: Updated close task endpoint to allow co-worker with permission to close project tasks they are not assigned to.

FIX: Updated update user endpoint to automatically create teams and settings for legacy accounts.

FIX: Fixed class cast exception in grant edit access endpoint.

FIX: Fixed constraint violation exception in update task endpoint.

FIX: Fixed an issue in Microsoft Signin when the user has no last name set up in their microsoft account.

FIX: Fixed several Sentry issues.

13 July 2023
API v4.93.6
IMPROVEMENT: Improved sync tracking task endpoint to automatically stop tasks when the date is before the start date of the event.

IMPROVEMENT: Added admins and PMs to Websocket messages sent in add, update and delete event endpoints.

IMPROVEMENT: Fixed an issue in list assigned tasks endpoint where the max limit would be too high.

FIX: Fixed an issue in Microsoft Outlook Connector where the calendar sync would get stuck while reading the HTTP input stream.

FIX: Fixed and issue in Google Calendar Connector where calendar events would be deleted after an exception occurs.

IMPROVEMENT: Updated Google Calendar Connector to set max results for Calendar API calls.

IMPROVEMENT: Updated Tomcat sever configuration settings to improve handling of performance issues.

FIX: Fixed an issue in update user endpoint where the user would have neither settings nor a team relation.

FIX: Fixed a notifications issue due to JSON mapping exceptions caused by illegal characters.

FIX: Add escape non ASCII for notification deserialization.

FIX: Fixed several Sentry issues.

IMPROVEMENT: Removed update operations from the auth endpoint to improve performance and avoid locking issues.

IMPROVEMENT: Removed deprecated event tag operations.

FIX: Fixed NullPointerException escaping backslash character in Instant Updates cronjob.

FIX: Fixed an issue in generate timecard endpoint where extra hours would not be automatically detected.

NEW: Added localization key to default server responses when an error occurs to improve error management in Sentry.

FIX: Fixed an issue where events would get duplicated after a user unlinks and then re-links their calendar.

03 July 2023
API v4.93.5
FIX: Fixed database session closed issue in Microsoft Outlook Connector.

NEW: Added delete account endpoint in backoffice system to better comply with account deletion requests.

NEW: Added separate connection pool for Shiro transactions to try and avoid pool exhaustion issues.

IMPROVEMENT: Improved connection handling in update task endpoint to avoid using multiple connections.

FIX: Refactored update intercom feature to avoid storing repeated accounts in the webhooks queue.

FIX: Fixed drop failed queued calendar events (in processing state) to prevent duplication issues.

27 June 2023
API v4.93.4
IMPROVEMENT: Improved error handling in Trello Connector when user tokens are expired.

FIX: Refactored list assigned tasks queries to improve performance.

IMPROVEMENT: Removed deprecated list tasks endpoints.

FIX: Fixed several configuration issues with slf4j and Logback libraries.

NEW: Improved Microsoft Outlook and Google Calendar Connectors to include calendars' missing occurrences.

IMPROVEMENT: Added abandoned database connection parameters to Tomcat configuration.

NEW: Refactored Microsoft Outlook and Google Calendar Connectors to get calendar events from the last 5 years.

FIX: Fixed several connection leaks when using additional database connections.

16 June 2023
API v4.93.3
FIX: Refactored all cronjobs to avoid overwriting the exectute methods used by New Relic traces.

NEW: Added missing occurrences within sync range in Microsoft Calendar.

FIX: Added missing Microsoft Event Importer logs.

IMPROVEMENT: Updated batch-size of Google Helper Process cronjob to improve performance.

FIX: Fixed NullPointerException when calendar timezone is null.

IMPROVEMENT: Incremented Quartz threads count to avoid performance issues.

NEW: Updated cronjobs to send custom attributes to New Relic in all cronjobs.

FIX: Updated calendar cronjobs to use the correct method to normalize timezones.

IMPROVEMENT: Added taskService to search tasks endpoint.

IMPROVEMENT: Updated calendar cronjobs to avoid executing unnecessary queries many times.

FIX: Fixed Microsoft Calendar event retrieval infinite loop issue.

06 June 2023
API v4.93.2
FIX: Updated calendar cronjobs to delete master events after deleting occurrences.

FIX: Updated calendar cronjobs to reset version control to force sync after update.

FIX: Updated calendar cronjobs to update previous occurrences marked as deleted.

FIX: Fixed null pointer exception reading network stream from HTTP connections.

IMPROVEMENT: Updated batch sizes of Trello cronjobs to 20.

IMPROVEMENT: Updated logging implementation of user logs to log only webhook events that belong to accounts with an active webhook subscription.

IMPROVEMENT: Updated frequency of Webhooker cronjob to 200.

NEW: Added custom attributes to traces of Trello cronjobs.

01 June 2023
API v4.93.1
IMPROVEMENT: Added created_at and for all entries and modified frequency and batch size of Cookie Cleaner cronjob.

IMPROVEMENT: Updated subscribe endpoint to whitelist accounts for sending unlimited emails per day.

FIX: Fixed URL length issue in add dictionary term endpoint.

FIX: Fixed handle bar errors in response messages.

IMPROVEMENT: Added open / close task / project user logs for webhooks.

FIX: Fixed wrong method call in Microsoft Helper Process cronjob.

IMPROVEMENT: Improved label names in Zapier output endpoint.

24 May 2023
API v4.93.0
FIX: Fixed a cast exception issue in update holiday endpoint.

IMPROVEMENT: Updated webhooker cronjob batch_size and disallowed concurrent execution.

NEW: Added support for sending output and cronjob logs to NewRelic.

FIX: Fixed NewRelic logging in WebhookerDataProcessorJob.

IMPROVEMENT: Added logging.properties to configure logging settings.

FIX: Added batch size in calendar mapping cronjob.

IMPROVEMENT: Refactored batch processing of calendar mapping cronjob to implement a FIFO strategy.

NEW: Added NewRelic custom attributes for all cronjobs.

FIX: Excluded Shiro and Google API messages from output logs.

IMPROVEMENT: Improved all calendars processed message.

NEW: Added logs in Trello cronjob to calculate how long it takes to sync all Trello accounts.

IMPROVEMENT: Improved Calendar logging with multiple channels (Sentry, NewRelic, standard output, etc).

IMPROVEMENT: Added UserExceptions to Sentry.

FIX: Updated batch size of Microsoft Helper Process cronjob (200).

FIX: Fixed NPE in Calendar Mapping Syncher cronjob when a user disables one of their calendars.

FIX: Deleted debugging Trello logs.

18 May 2023
API v4.92.9
NEW: Added search projects endpoint for Zapier integration.

NEW: Added search tasks endpoint for Zapier integration.

NEW: Added search events endpoint for Zapier integration.

NEW: Added create input endpoint for Zapier integration.

FIX: Updated expired Microsoft app secret.

IMPROVEMENT: Updated default console logging for Shiro, EasyRules and Google API libraries to avoid recording unnecessary logs.

FIX: Fixed null pointer exception when autolog_events parameters in update calendar endpoint is null.

FIX: Deleted debugging logs for oldThirdPartyProject not null in Trello Connector.

FIX: Fixed an issue where microsoft calendar would not update task of synced events.

FIX: Fixed an issue with null private events in Calendar delegate.

FIX: Fixed API rate limit issues in Trello Connector.

NEW: Added custom attributes in New Relic cronjob traces.

10 May 2023
API v4.92.8
FIX: Fixed an issue on the v3 of the button where timezone would be encoded incorrectly.

IMPROVEMENT: Added support for sending JSON in the body of POST requests.

FIX: Fixed issue with debug emails in Stripe's card created webhook.

IMPROVEMENT: Added webhook action to validate total accumulated time on task users.

NEW: Added support for deleting third party events when an event, task or project has been deleted.

NEW: Added support for default tasks linked to a calendar.

NEW: Added support for private events in calendars.

IMPROVEMENT: Updated close task and close project endpoints to allow users to close objects imported from Trello.

NEW: Updated update task endpoint to avoid un-assigning users from calendar sync tasks.

FIX: Added list set custom fields limit of 20.000 in list events endpoint.

IMPROVEMENT: Improved error logging in Sentry in Microsoft Connector.

FIX: Fixed authentication issue in Connect endpoints where archived users could still access some endpoints.

FIX: Updated list trackables in Trello to avoid showing closed tasks.

IMPROVEMENT: Improved Stripe cronjob to avoid processing failed subscription updates (max retry count).

IMPROVEMENT: Added timezone support for last_requested_at parameter in calendars.

IMPROVEMENT: Added created_at parameters in sync helper and third party event mapping.

FIX: Fixed email preference validation issue when sending invoicing and timeoff emails.

NEW: Added support for re-syncing calendars when the linked calendar sync task is updated.

FIX: Fixed logging issues with EasyRules library.

FIX: Fixed email issues in update task, close task, and close project endpoints.

IMPROVEMENT: Improved logging in Trello cronjobs.

IMPROVEMENT: Implemented Logback library for improved logging.

19 April 2023
API v4.92.7
IMPROVEMENT: Added DisallowConcurrentExecution annotation to EventJob in calendar cronjobs.

IMPROVEMENT: Updated frequency and batch size of Trello cronjobs.

IMPROVEMENT: Improved error handling and invoice auto-charge in Stripe's card created webhook.

FIX: Fixed duplicate issue in Trello's app task management settings when turning the app on.

FIX: Added batch size for revert processing status in calendar cronjobs.

IMPROVEMENT: Added filter (active, archived, all) to list customers endpoint.

11 April 2023
API v4.92.6
NEW: Added CRUD endpoint for managing third party custom fields.

IMPROVEMENT: Added DisallowConcurrentExecution annotation to Trello cronjobs.

FIX: Fixed null third_party_mapping_id in calendar cronjobs.

FIX: Fixed auto-forwarding in deprecated List Events endpoint.

IMPROVEMENT: Refactored accept invite endpoint using multiple transactions to avoid inconsistent team status update.

NEW: Added search projects endpoint.

NEW: Added search tasks endpoint.

NEW: Added search events endpoint.

FIX: Fixed duplicate issue with wrong status code when inviting the same user twice.

FIX: Fixed duplicated events issue in calendar cronjobs.

IMPROVEMENT: Refactored process card created webhook to support different webhook types. Deprecated legacy add card form. Now supporting only updates from Customer Portal and invoice links.

31 March 2023
API v4.92.5
IMPROVEMENT: Improved error logging in calendar cronjobs.

IMPROVEMENT: Added multi-instance support for cronjobs logging.

IMPROVEMENT: Updated calendar cronjobs to find previous calendar events in memory reducing DB queries.

FIX: Fixed update status (open / closed) in Trello Task Importer.

IMPROVEMENT: Improved error handling for Stripe card created webhook.

IMPROVEMENT: Improved matching algorithms in new search projects and search tasks endpoints.

27 March 2023
API v4.92.4
NEW: Added new search projects endpoint.

NEW: Added new search tasks endpoint.

NEW: Added new search events endpoint.

FIX: Fixed Istanbul time zone issue in Microsoft Outlook Connector.

NEW: Added webhook to handle Stripe card updates.

IMPROVEMENT: Added logs for processed calendars.

IMPROVEMENT: Updated calendar cronjobs to avoid updating last sync at parameter when calendar is DROPPED.

IMPROVEMENT: Updated calendar cronjobs to only retrieve events up to 3 months in the future.

IMPROVEMENT: Added update calendar endpoint validations for required parameters.

IMPROVEMENT: Added Sentry logs to all Stripe webhooks.

IMPROVEMENT: Removed active subscription constraint to allow users with canceled subscriptions to access their Customer Portal.

21 March 2023
API v4.92.3
FIX: Fixed a string length issue in add and update holiday endpoints to allow users to select more users.

FIX: Updated missing modified instances of events in calendar cronjobs.

IMPROVEMENT: Refactored legacy endpoints to forward them to the new ones.

IMPROVEMENT: Added Sentry logs to Workplan cronjobs.

IMPROVEMENT: Added support for customer names in account search endpoint.

FIX: Fixed an issue with Instanbul IANA timezone in Microsoft calendar cronjobs.

15 March 2023
API v4.92.2
IMPROVEMENT: Updated New Relic agents in all instances.

FIX: Fixed multi-account issue in accept invite endpoint where confirmation emails to inviters would not be sent.

FIX: Fixed duplicate issue in list project users endpoint.

FIX: Fixed delete others' events when users unlink their calendars.

FIX: Fixed mark BLOCKED third-party events with null event ID as DELETED.

FIX: Fixed an issue in Google and Microsoft Calendar cronjobs when syncing event instances.

09 March 2023
API v4.92.1
IMPROVEMENT: Added hostname to calendar cronjob logs and updated MS mapping frequency to 1 minute.

IMPROVEMENT: Added New Relic instrumentation for cronjobs.

IMPROVEMENT: Refactor calendar cronjobs to check for update before processing calendars.

IMPROVEMENT: Updated subscribe_cancel_url in Stripe checkout with the release of the new Business plan.

FIX: Fixed an issue in calendar cronjobs where events would be incorrectly deleted.

27 February 2023
API v4.92.0
FIX: Fixed several issues in calendar cronjobs.

IMPROVEMENT: Updated payment failed webhook to avoid sending 500 responses when an account is not found.

FIX: Fixed queries in Trello cronjobs to reduce the number of user accounts to be processed.

FIX: Fixed an issue in get project endpoint when request third party projects.

FIX: Fixed NPE in resend user invite endpoint.

IMPROVEMENT: Updated batch size and frequency of Microsoft Outlook cronjobs to improve performance.

23 February 2023
API v4.91.10
IMPROVEMENT: Added group calendar notifications.

FIX: Fixed an issue when adding default events in the future with different validity periods.

FIX: Fixed foreign key in event_id column in third party event mapping table.

IMPROVEMENT: Deprecated legacy list projects endpoint. All requests are forwarded to the new version.

FIX: Fixed an issue in Workplan (Daily) cronjob where users would be incorrectly notified.

IMPROVEMENT: Added logs when calendars which are ok are marked as pending.

17 February 2023
API v4.91.9
FIX: Fixed an issue in list invoices endpoint where the account logo would not be included in the response.

IMPROVEMENT: Added last requested_at and pending_sync_events and last_failed_request_at parameters to list calendars endpoint to inform users about the progress of the sync process.

FIX: Fixed an issue in login endoint when using app passwords.

IMPROVEMENT: Added email notification to sync calendar endpoint when the initial synchronization is completed.

IMPROVEMENT: Updated frequency and batch size of Google Calendar cronjobs to improve performance.

NEW: Added sync Trello endpoint to trigger an initial synchronization.

IMPROVEMENT: Updated calendar cronjobs to save revoked token email notification in the queue.

FIX: Added Trello app in list apps endpoint.

FIX: Fixed null pointer exception when validating users email preferences.

FIX: Fixed an issue in calendar cronjobs where deleted events would automatically be added again.

FIX: Fixed several issues in websocket messages when projects are added or updated.

FIX: Fixed a NPE in close projects in batch endpoint.

08 February 2023
API v4.91.8
FIX: Fixed an issue in add repeating events with frequency 0 that would cause an infinite loop.

FIX: Fixed error messages in tracking policies.

IMPROVEMENT: Improved logging in Connect Syncer cronjob.

NEW: Added grouped subtask notifications.

IMPROVEMENT: Added avoid_workspaces column in Trello settings, so that users can specify workspaces to avoid syncing.

IMPROVEMENT: Added trello_legacy column in Task Management settings to identify Button users that work with Trello.

FIX: Fixed session handling in calendar cronjobs to avoid leaving database sessions idle.

IMPROVEMENT: Added websocket messages to Connect Syncer cronjob.

FIX: Fixed a policy issue in duplicate task endpoint when projects were required.

FIX: Fixed null pointer exception in list user timecards endpoint.

FIX: Fixed total time off count in list user timecards endpoint.

FIX: Removed archived tasks from list third party tasks endpoint.

IMPROVEMENT: Updated delete time off endpoint to allow admins of certain accounts to delete imported time offs.

IMPROVEMENT: Added different environments for Trello Sync.

FIX: Fixed an issue in accept invite endpoint where the status would remain as invited when users signed in with their Apple or Microsoft accounts.

IMPROVEMENT: Updated default not found message across all API endpoints.

FIX: Fixed an issue in list events with time offs where part-time time offs would not be included in the response.

20 January 2023
API v4.91.7
FIX: Fixed list events min endpoint with task filter.

IMPROVEMENT: Added account id in auth endpoint's response.

IMPROVEMENT: Added more database exceptions to Sentry.

FIX: Fixed an issue in update task endpoint where a ghost task would remain in a invalid state with hidden visibility.

IMPROVEMENT: Improved error logs in Trello cronjobs.

FIX: Fixed an issue in generate user timecard endpoint where all-day time offs would be incorrectly displayed.

10 January 2023
API v4.91.6
FIX: Fixed project issue in add event endpoint with third party service.

IMPROVEMENT: Added qr code to add invoice endpoint.

FIX: Fixed multi-account issue in list apps endpoint.

IMPROVEMENT: Improved error handling for Trello API responses with codes: 503, 504 and 429.

IMPROVEMENT: Refactored Trello cronjobs naming for the sake of consistency.

FIX: Fixed an issue when retrieving project null in get project endpoint.

IMPROVEMENT: Added third party data in list user tasks endpoint.

NEW: Added duplicate task endpoint.

NEW: Added grouped task notifications in close task endpoint.

NEW: Added grouped task notifications in re-open task endpoint.

FIX: Fixed null pointer exception in delete timecard days in batch endpoint.

23 December 2022
API v4.91.5
FIX: Fixed pagination issue in list events endpoint.
22 December 2022
API v4.91.4
FIX: Fixed exception in user logs when saving array of events.

FIX: Fixed log user events update.

FIX: Fixed get project endpoint by third party service id.

FIX: Fixed an issue with user permissions in event and project webhooks.

IMPROVEMENT: Refactored list min events query to improve performance.

FIX: Fixed an issue in SAP export cronjob to avoid including events linked to deleted projects.

IMPROVEMENT: Added from and to parameters to start SAP Export cronjob endpoint to specify a period for exporting events.

IMPROVEMENT: Updated webhook implementation to set status = DROPPED for user logs without valid strategy.

17 December 2022
API v4.91.3
FIX: Fixed an issue in verify user email by deleting the port 8080 from the call-to-action URL.

IMPROVEMENT: Added validation to SAP export cronjob to limit the number of chars in event notes to 40.

FIX: Fixed an issue in export timecards where users would not receive the email.

FIX: Fixed an issue in Trello Connector to automatically update task status (open / closed).

NEW: Added project webhooks support for Zapier.

FIX: Fixed NPE in track task with Trello.

FIX: Fixed NPE in add event with Trello.

FIX: Fixed NPE in update project estimated time with Trello.

12 December 2022
API v4.91.2
FIX: Fixed Bamboo connector issue to update time offs in TrackingTime that have been canceled in Bamboo.

FIX: Fixed null pointer exception in get activities summary.

FIX: Fixed property issue in Factorial that caused the Oauth process to fail.

FIX: Fixed class cast exceptions in user logs.

FIX: Fixed Stripe pricing issue for new annual plans.

IMPROVEMENT: Added description to task user logs.

03 December 2022
API v4.91.1
FIX: Fixed sizes of default account logos.

IMPROVEMENT: Improved error handling in Stripe checkout.

NEW: Added task webhooks.

01 December 2022
API v4.91
NEW: Added Zapier integration for add and update time entries webhook events.

FIX: Fixed custom fields in SAP integration.

FIX: Fixed issues with partial time offs in Bamboo Connector.

FIX: Fixed issues in Trello Connector that broke the OAuth 1.0 implementation.

17 November 2022
API v4.90.7
NEW: Added ping endpoints for new cronjobs.

FIX: Removed user roles validation in add and update time off endpoints.

IMPROVEMENT: Added created_at column in Stripe queue.

FIX: Fixed several issues in Factorial connector before GA release.

IMPROVEMENT: Updated Bamboo Connector to better map partial time offs (days and hours)

10 November 2022
API v4.90.6
IMPROVEMENT: Added websocket messages to update work schedule endpoint.

FIX: Added user logs in add event endpoint.

NEW: Added endpoint to delete users with status invited.

FIX: Added Basecamp properties in prod environment.

FIX: Fixed email template for workplan (weekly) cronjob.

NEW: Added groupped email notifications in assign tasks endpoints.

FIX: Fixed project estimated times in SAP cronjob.

FIX: Updated SAP import cronjob to automatically archive projects that are not present in the FTP file.

FIX: Fixed a default events validation issue in add event endpoint.

FIX: Fixed several issues in assign projects where users have not accepted their invitations yet.

03 November 2022
API v4.90.5
FIX: Fixed an issue in multiple unassign tasks action which would produce a null object id.

IMPROVEMENT: Removed test accounts from Intercom updates.

IMPROVEMENT: Improved Webhooks Queue Cleaner cronjob to clean dropped objects.

IMPROVEMENT: Updated duplicate project endpoint to send emails to project members when the option keep assignees is selected.

IMPROVEMENT: Updated sync calendar endpoint to send a websocket message when the process is finished.

IMPROVEMENT: Improved error messages in login endpoint when the user has been invited but has not yet verified their account.

IMPROVEMENT: Updated reminders cronjobs to notify admins when a user hasn't accepted their invitation.

FIX: Fixed null pointer exception in assign users to work schedule endpoint.

21 October 2022
API v4.90.4
FIX: Fixed email notifications in assign tasks in batch endpoint.

IMPROVEMENT: Refactored checkout and customer portal endpoints to authenticate requests using tokens.

20 October 2022
API v4.90.3
FIX: Fixed translation issue in instant update scheduler to always default to EN.
19 October 2022
API v4.90.2
FIX: Fixed issues in instant update scheduler.
18 October 2022
API v4.90.1
FIX: Fixed issues in instant update scheduler.
17 October 2022
API v4.90
NEW: Added support for account logo in Invoicing add-on.

FIX: Fixed an issue in the close project endpoint where archived users would still be notified.

IMPROVEMENT: Refactored assign tasks emails to use new webhook architecture and avoid sending multiple emails.

IMPROVEMENT: Updated archive user endpoint to automatically turn off calendar synchronization if any.

NEW: Added activity summary endpoints for Autotrack add-on.

FIX: Fixed notification messages in new English translation.

IMPROVEMENT: Added missing translations for API messages in ES, DE, FR and PT.

IMPROVEMENT: Refactored URL for starting and stopping Workplan (Weekly) scheduler.

IMPROVEMENT: Refactored weekly report with new template in SendGrid.

IMPROVEMENT: Refactored SAP integration features according to v2 specifications.

FIX: Removed status superceded from Bamboo Connector, so that only approved time offs will be synced.

FIX: Fixed several issues in list events endpoint when rendering time off and holiday events.

IMPROVEMENT: Updated archive user endpoint to set closed_at and closed_by metadata attributes.

NEW: Added new Autotrack user add-on.

FIX: Fixed an issue in Bamboo cronjobs where time off from archived users would still be synced.

FIX: Fixed an issue in list assigned tasks endpoint where session user is of role coworkers where some projects would appear duplicated.

IMPROVEMENT: Activated user logs for free accounts.

FIX: Fixed an issue in import events endpoint where new events would be associated to archived users.

FIX: Fixed an issue in update tasks in batch endpoint where the value of custom fields would be deleted.

20 September 2022
API v4.90.7
FIX: Removed automatic dictionary addition in localization manager to avoid database connection leaks.

IMPROVEMENT: Updated add and update time off endpoint to evaluate public holidays when counting days.

FIX: Fixed an issue in english error messages where handlebars would not be replaced.

IMPROVEMENT: Added plan type "flexible" to allow new customers to buy more seats than the current number of active users in their account.

IMPROVEMENT: Added email categories to analyze open and click rates in Sendgrid.

09 September 2022
API v4.89
NEW: Refactored localization methodology.

IMPROVEMENT: Added app_attendance, tasks count, schedules and app_calendar_ms parameters to Intercom.

IMPROVEMENT: Removed system custom fields from custom fields count in Intercom.

IMPROVEMENT: Updated Bamboo sync endpoint to make the employee id custom field editable.

FIX: Fixed exception in user logs when the IP address would be null.

IMPROVEMENT: Updated update account endpoint to allow clients to remove the account logo and added logo_url to account JSON at login.

FIX: Fixed name validation in public holidays to allow users to add holidays with the same name for different countries.

FIX: Fixed timecard policies to allow users to submit days for the whole month.

IMPROVEMENT: Improved error message in login endpoint when a user with a given email doesn't exist.

FIX: Fixed issue in signin with Microsoft endpoint where the user name would be null.

IMPROVEMENT: Added websocket message in sync calendar endpoint.

IMPROVEMENT: Updated cleanse project processor to update accumulated time of tasks.

FIX: Fix an issue in switch plan endpoint where coupons would not be deleted.

IMPROVEMENT: Added seat validation when subscribing to the freelancer plan.

FIX: Fixed signin with apple issue.

IMPROVEMENT: Updated calendar synchronization cronjobs to avoid processing data from archived users.

10 August 2022
API v4.88
NEW: Added timecard days generator endpoint for migrating users to the new timecards.

FIX: Fixed null pointer exception in add time off endpoint.

IMPROVEMENT: Added translations for default time off types.

IMPROVEMENT: Updated frequency for SAP integration schedulers.

FIX: Fixed an issue in the Timecap scheduler to avoid sending repeated emails.

FIX: Fixed user logs in update user endpoint.

FIX: Fixed an issue in the workplan scheduler where account schedules could not be verified.

IMPROVEMENT: Added more detailed Sentry logs for the SAP integration.

25 July 2021
API v4.87
FIX: Updated export parameters in SAP integration.

FIX: Fixed export file name in SAP integration.

IMPROVEMENT: Added support for validity periods in workplan cronjobs (daily and weekly).

FIX: Fixed OAuth URL for Microsoft signins.

FIX: Fixed issue in list user timecards where updating a user's hiring date would not be taken into consideration.

IMPROVEMENT: Added timezone support in get task log endpoint.

IMPROVEMENT: Added support for muting Sentry logs by severity level.

FIX: Fixed issues in worklplan (weekly) cronjob.

FIX: fixed an issue in list tasks endpoint where the working on parameters would be missing.

IMPROVEMENT: Added support for timeoffs and holidays in workplan cronjobs.

NEW: Added endpoint to delete websocket device tokens.

NEW: Added allow_editing_days_threshold parameter in update app timecard settings endpoint.

IMPROVEMENT: Added locale, timezone and timezone_iana parameters in signup, accept user invite and verify user token endpoints.

IMPROVEMENT: Improved error messages when lock acquisition exceptions arise.

IMPROVEMENT: Updated third party signin endpoints to retrieve the user's name and surname.

FIX: Fixed an issue in list events endpoint where work events whould be displayed along with time off events.

FIX: Fixed an issue in sync bamboo endpoint where archived users would also by synced.

FIX: Fixed an issue in get integration endpoint where the bamboo add-on would only be displayed to the user who turned it on.

IMPROVEMENT: Removed time off notes from Bamboo sync endpoint and cronjob.

NEW: Added timecard generator endpoint to migrate timecards to v2.

FIX: Fixed status issues in add time off endpoint.

NEW: Added SFTP implementation for SAP integration.

28 June 2022
API v4.86
FIX: Fixed an issue in list projects for co-worker users where the is_public parameter would always be null.

NEW: Added Intercom integration in add task comment endpoint.

FIX: Fixed a billing issue in events directly linked to projects.

FIX: Fixed an issue when granting edit access by month.

23 June 2022
API v4.85
IMPROVEMENT: Added new localization implementation and translatable keys.

NEW: Refactored list tasks endpoint to improve response times and size.

IMPROVEMENT: Added new redirect URLs in new Stripe checkout.

IMPROVEMENT: Updated timecard days statuses.

IMPROVEMENT: Added tax id parameter in new Stripe checkout.

IMPROVEMENT: Updated turn on timecard app to set needs approval parameter to false by default.

IMPROVEMENT: Added default system custom fields hiring date for all accounts.

NEW: Added delete timecard days endpoint.

13 June 2022
API v4.84
FIX: Fixed redirect URLs in new Stripe checkout.

FIX: Fixed tasks email template to display the direct link again.

IMPROVEMENT: Added Sentry logs to new Stripe checkout.

NEW: Added support for Business and Freelancer plans.

IMPROVEMENT: Added status parameter to save timecard day endpoint.

IMPROVEMENT: Refactored implementation of system custom fields so that free accounts can use them.

FIX: Fixed an issue in get next number endpoint where the response would always be zero.

IMPROVEMENT: Updated default day status in generate user timecard endpoint.

07 June 2022
API v4.83.1
FIX: Updated list tasks endpoint to update the max size to 1000.

FIX: Fixed New relic property to include all transactions in APM.

NEW: Added new Stripe plans.

03 June 2022
API v4.83
FIX: Fixed issue in list invoices endpoint where the invoice status would be incorrectly displayed as overdue.

IMPROVEMENT: Added support for authenticating with app passwords without email.

IMPROVEMENT: Refactored get project endpoint to forward requests to the optimized new version.

IMPROVEMENT: Added logs and reduced max list size in list tasks endpoint.

IMPROVEMENT: Refactored custom field endpoints to allow free users to edit system custom fields.

FIX: Fixed performance issues in update timeoff endpoint by adding indexes to cookies table.

IMPROVEMENT: Added support for IANA timezones to avoid daylight saving issues.

IMPROVEMENT: Added a third option when turning off calendar synchronization to allow users to keep events already linked to a project / task.

FIX: Fixed an issue in save timecard day endpoint where the type would not be saved.

NEW: Added support for temporary edit access in save timecard day endpoint.

FIX: Fixed null pointer exception in list team timecards where users would not have any metadata.

IMPROVEMENT: Add new day status in timecards to identify entries submitted by users.

FIX: Fixed an issue in get work schedule where users would be missing.

IMPROVEMENT: Updated frequency of Bamboo cronjobs to run once per day.

FIX: Fixed an issue in list events endpoint where timeoffs for a given user would not be included in the response.

NEW: Added support for default events in generate user timecard endpoint.

FIX: Fixed an issue in get user timecard endpoint where the user would be created within the last days.

IMPROVEMENT: Updated list timeoff types endpoint to sort objects by name.

IMPROVEMENT: Improved naming of default timeoff types.

IMPROVEMENT: Improved queries to retrieve accounts in Timecard Reminder cronjob.

IMPROVEMENT: Improved queries to retrieve accounts in Timecard Generator cronjob.

IMPROVEMENT: Added default get timeoffs since in Bamboo and Factorial settings.

IMPROVEMENT: Updated delete work schedule endpoint to block deletion in case assignments with validity periods exists.

IMPROVEMENT: Improved error messages in schedule endpoints.

IMPROVEMENT: Updated Factorial time off types cronjob to mark all types as approvable by default.

FIX: Fixed an issue in list work schedules endpoint where a user who has two schedule assignments in different periods would appear twice in the response.

IMPROVEMENT: Updated list schedules endpoint to not include archived users.

IMPROVEMENT: Updated list schedules endpoint to return users always in the same order.

NEW: Added time off events in list events endpoint for holidays when the add-on default events is turned on.

NEW: Added export timecards endpoint to export in PDF format and zipped.

IMPROVEMENT: Added company details when exporting timecards in PDF.

IMPROVEMENT: Refactored delete project endpoint to improve response times.

IMPROVEMENT: Refactored Stripe cronjob logging to distinguish between errors and warnings.

IMPROVEMENT: Added payments parameter in get invoice endpoint.

FIX: Fixed overdue issue in list invoices endpoint.

13 May 2022
API v4.82
IMPROVEMENT: Added discount parameter to invoice item.

NEW: Added open tasks in batch endpoint.

NEW: Added close tasks in batch endpoint.

FIX: Fixed button last seen user metadata.

FIX: Fixed an issue in list invoices endpoint where paid invoices would be incorrectly marked as overdue.

FIX: Fixed an issue in update project endpoint where user_index would allways be missing.

FIX: Fixed websocket message body for task closed and task opened events.

FIX: Fixed timezone normalization issue.

06 May 2022
API v4.81
NEW: Added new localization system to translate API copy in multiple languages.

IMPROVEMENT: Added default status parameter in CTA objects.

FIX: Fixed an issue in update event endpoint where the accumulated times of projects would be wrong when converting events from scheduled to logged.

FIX: Fixed account status value in auth endpoint.

NEW: Added system custom fields for projects timeline.

FIX: Fixed a custom field issue where you could link a field of a given type to an object of another type.

FIX: Fixed null pointer exception in cleanse project cronjob.

FIX: Fixed null pointer exception in payment failed webhook when updating Intercom.

IMPROVEMENT: Fixed updating user metadata last seen in button.

IMPROVEMENT: Added new invoice parameters, default tax rates, payment terms and more.

FIX: Fixed timezone issue in task's metadata parameters created_at and updated_at.

28 April 2022
API v4.80
IMPROVEMENT: Updated all endpoints to always include custom fields in the response by default.

IMPROVEMENT: Updated Reminder cronjob to send two reminders to users who haven't accepted their invites instead of three.

NEW: Added update tasks in batch endpoint.

NEW: Added update events in batch endpoint.

IMPROVEMENT: Updated auth endpoint to include account data.

IMPROVEMENT: Improved response times in the list docs endpoint by removing the doc data.

NEW: Updated update user endpoint to allow users to delete their profile image.

IMPROVEMENT: Added churned_at parameter to Intercom cronjob.

FIX: Fixed timezone issue in task endpoints. Now the timezone for the created_at and updated_at parameters are correctly converted to the account's timezone.

IMPROVEMENT: Refactored Accountant cronjob to use multiple transactions instead of one to avoid database deadlock issues.

NEW: Added data cleanse cronjob processors and synchers to fix inconsistencies in project's accumulated times.

FIX: Updated add subscription endpoint to change Stripe's payment behavior and avoid incomplete and expired subscriptions.

IMPROVEMENT: Added S3 support for SAP integration cronjobs.

IMPROVEMENT: Added automatic source and destiny file rotation in SAP integration cronbjos.

FIX: Fixed an issue in update task endpoint where a task would be linked to a task list that belonged to a task list from another project.

FIX: Refactored sync tracking task endpoint to always retrieve the same task_user object and avoid tracking conflicts.

NEW: Added new backoffice endpoint to import projects and default tasks in batch from an Excel file.

IMPROVEMENT: Added new parameters to invoice settings.

FIX: Fixed an issue in search projects endpoint where the response would be incorrectly formatted in case the search key is too short.

NEW: Added new backoffice endpoint to import users in batch.

29 March 2022
API v4.79
NEW: Updated Stripe library to the latest version.

NEW: Added new Stripe Checkout.

NEW: Added Stripe portal endpoint.

NEW: Added list Stripe receipts endpoint.

NEW: Added list Stripe invoices endpoint.

FIX: Fixed date and parameter naming format in list system logs endpoint.

FIX: Fixed logs when starting schedulers.

FIX: Fixed Stripe plan IDs for Audit Trail add-on.

FIX: Fixed Stripe scheduler's update when the account is subscribed to the Audit Trail add-on.

NEW: Added check calendar status endpoint to verify whether a calendar is currently being synced or not.

IMPROVEMENT: Improved logs for Calendar schedulers.

IMPROVEMENT: Added company details to Timecards settings.

IMPROVEMENT: Removed times from edit access emails.

IMPROVEMENT: Improved password security. Now, at least 8 chars are required.

IMPROVEMENT: Added task description, project notes and subtasks when duplicating projects in add project endpoint.

FIX: Fixed iCal subscription links for multi-account users.

NEW: Added new weekly report scheduler using new email template and the sync queue.

IMPROVEMENT: Removed all-day events from Outlook and Google Calendar synchronization schedulers.

IMPROVEMENT: Improved logging in Factorial and Bamboo schedulers.

NEW: Added SAP Integration cronjobs.

IMPROVEMENT: Added account prompt in Microsoft Signin consent screen to ask users to choose an account.

IMPROVEMENT: Improved error messages from third party API in BasecampConnector.

09 March 2022
API v4.78
FIX: Updated cancel subscription endpoint to change the default team in case the user has other accounts.

IMPROVEMENT: Added system logs when schedulers are started and stopped.

NEW: Added app passwords for third party API authentication.

FIX: Fixed auto stop timers in Timecop cronjob.

FIX: Fixed cron schedule endpoint to send weekly reminders for work schedules.

IMPROVEMENT: Added sort projects by user endpoint.

IMPROVEMENT: Added default custom fields order, so that they are shown consistently across apps.

FIX: Fixed email message in revoke edit access endpoint.

03 March 2022
API v4.77
FIX: Fixed multi-account issue in iCal subscription endpoint.

IMPROVEMENT: Improved error messages in new policies.

NEW: Added block editing events weekly and monthly to policies.

IMPROVEMENT: Added user email to support access granted email.

FIX: Fixed frequency in new weekly report scheduler.

22 February 2022
API v4.76
NEW: Added Jira Connector.

NEW: Added timeoff events based on schedule default events.

IMPROVEMENT: Added block_events_editing_monthly and block_events_editing_weekly to policies.

IMPROVEMENT: Added required_browser_extension and require_desktop_app to policies.

IMPROVEMENT: Added allow_tracking_days_future_threshold to policies.

FIX: Fixed null pointer exception in list timeoff events for free accounts.

IMPROVEMENT: Added filter parameter in list teams endpoint.

IMPROVEMENT: Added order_by_closed_at parameter in list projects endpoint.

IMPROVEMENT: Added app_url and metadata parameters in add and list activity logs endpoints.

NEW: Added grant support access.

NEW: Refactored cronjob to send weekly email reports.

IMPROVEMENT: Added account filter to calendar cronjobs to avoid processing calendars from free accounts.

FIX: Fixed issues in Budgeter cronjob.

IMPROVEMENT: Added exception in sync calendars endpoint to avoid processing calendars in parallel.

IMPROVEMENT: Added third party mapping id in sync helper tables.

IMPROVEMENT: Added user logs in calendar endpoints.

IMPROVEMENT: Added missing cronjobs to start schedulers endpoint.

FIX: Fixed issues in duplicate project endpoint.

FIX: Fixed extension missing issue in download file endpoint.

NEW: Added list system logs endpoint.

IMPROVEMENT: Added user logs with metadata in delete events endpoint.

IMPROVEMENT: Added subscription reference validation in update account backoffice endpoint.

IMPROVEMENT: Localized dates in timeoff endpoints.

IMPROVEMENT: Added time offs to timecards.

IMPROVEMENT: Added is workable verification in the server backend in save timecard day endpoint.

12 January 2022
API v4.75
FIX: Fixed list time offs when from and to are on the same date.

IMPROVEMENT: Improved error logging for canceled and suspended accounts.

NEW: Added endpoint mark CTA as undone.

FIX: Fixed policy issue in track third party task endpoint.

IMPROVEMENT: Added logging in backoffice update account endpoint.

NEW: Refactored weekly email reports.

NEW: Added Timecard Generator cronjob.

NEW: Added grant support access.

NEW: Added new policy parameters.

NEW: Added user custom field for employee id in Bamboo Connector.

NEW: Added auto stop timers in Timecop cronjob./li>

IMPROVEMENT: Added timecard system custom field start_date to indicate when a user was hired.

NEW: Added grant edit access to allow users to edit their time entries.

IMPROVEMENT: Refactored task syncher cronjob to support tasks linked to multiple projects.

FIX: Fixed null pointer exception for free accounts in upload file endpoint.

FIX: Updated Log4J library to v2.7.1.

FIX: Fixed cast issue in Factorial Connector's sync users endpoint.

23 December 2021
API v4.74
FIX: Updated webhooks queue Intercom cronjobs (processor and syncer) to avoid creating duplicated entries.

IMPROVEMENT: Added third party data to tasks in list user assigned tasks endpoint.

IMPROVEMENT: Added batch size properties for webhooks Intercom cronjobs.

FIX: Fixed null pointer exception in update company in add subscription endpoint.

FIX: Updated Log4J library to patch vulnerability issue Log4Shell.

NEW: Added Timecard Reminder cronjob.

IMPROVEMENT: Added service_project_id in get task endpoint.

IMPROVEMENT: Updated update task and update project endpoints to ignore parameters that cannot be updated when objects are linked to a third party service.

IMPROVEMENT: Improved logging in webhook Intercom cronjobs.

FIX: Removed CTAs creation from login endpoint.

NEW: Added list CTAs endpoint.

IMPROVEMENT: Added Sentry logs to Stripe webhooks.

FIX: Fixed smart project selection for third party task management apps.

FIX: Fixed null pointer exception in accumulated and estimated times in import third party projects endpoints.

IMPROVEMENT: Added retry mechanism to webhook Intercom cronjobs to retry processing and syncing events.

FIX: Updated format of hour parameters in update intercom company to use doubles instead of strings.

IMPROVEMENT: Updated processing of gravatar and user initials lambda function to fail silently and fall back to default values in case those endpoints are unreachable.

NEW: Added user authentication endpoint.

FIX: Fixed user permissions in add account endpoint.

FIX: Fixed issue in list events min endpoint where time off events would not be included in the response when requesting a single day period.

FIX: Added back update intercom company requests on demand.

14 December 2021
API v4.73
FIX: Fixed null issue in estimated and accumulated times for imported projects.

IMPROVEMENT: Refactored transactional Intercom updates using the webhooks queue.

NEW: Added default time off types for TrackingTime add on.

FIX: Fixed null pointer exception in get project min endpoint.

NEW: Added import Intercom companies endpoint to update companies on demand using the webhooks queue.

NEW: Implemented webhooks queue and data model to update data on third party services on demand.

NEW: Added CTAs to show users new features and improvements.

IMPROVEMENT: Added url parameter in dictionary endpoints.

NEW: Added generate user timecard endpoint with time offs and holidays.

NEW: Added time off, time off types and holidays to save timecard day endpoint.

IMPROVEMENT: Added third party data to get project min endpoint.

NEW: Added service_project_id to track task, add event and update event endpoint, so that we can support tasks assigned to multiple projects in third party services.

FIX: Fixed Log4J vulnerability issue by upgrading the library to v2.16.0.

FIX: Fixed duplicate issue in intercom webhook update.

IMPROVEMENT: Added batch size properties for intercom data processor cronjob.

06 December 2021
API v4.72.1
FIX: Fixed an issue in get project min endpoint where co-workers' project assignments would not be returned.

IMPROVEMENT: Updated list events min endpoint to calculate time off events based on the users schedules.

FIX: Fixed account status update in cancel account endpoint.

IMPROVEMENT: Updated cancel account endpoint to automatically make other accounts the default team.

01 December 2021
API v4.72
FIX: Fixed an issue in get project min endpoint where the service would not be included in the server response.

IMPROVEMENT: Added Sentry logs for authentication errors (401) coming from our API clients.

FIX: Fixed save project preferences. Now, default timeline view can be updated.

IMPROVEMENT: Improved several error messages.

FIX: Fixed error in tracking policies where the option prevent users from tracking overtime was inverted.

FIX: Fixed an issue in Calendar cronjobs where calendar and instances IDs would exceed the max allowed char size.

FIX: Fixed an issue in export data endpoint where tasks and events would be limited. Now, all records are safely exported using pagination.

FIX: Fixed an issue in list project times endpoint where third party projects would return duration null instead of cero.

FIX: Fixed an issue that was causing deleted tasks not to be stored in trash.

FIX: Fixed expand series issue in Microsoft Calendar cronjobs.

FIX: Added third party sync status for third party projects.

IMPROVEMENT: Added created_at parameter in list filtered tasks endpoint.

24 November 2021
API v4.71
FIX: Fixed an issue in close task endpoint to include the task description in the server response.

IMPROVEMENT: Updated hibernate mappings for calendar tables to avoid deadlock issues.

IMPROVEMENT: Refactored workplan cronjob to avoid deadlock issues.

IMPROVEMENT: Added default_password, send_email and invite_link parameters to invite users in batch endpoint.

FIX: Fixed an issue in weekly email reports where not logged events would be included in the hour stats.

FIX: Fixed an issue with the account currency in the login endpoint. Now, the correct account currency of the default team is shown in the response.

IMPROVEMENT: Added app type in login endpoint's response.

FIX: Fixed duplicates issue in project docs. Now, all projects should only have one doc linked to them.

IMPROVEMENT: Added permalinks for projects and tasks in Basecamp and Trello connectors.

NEW: Added custom fields support in Asana connector.

FIX: Fixed multiple calendar issues.

FIX: Fixed import Asana tasks to retrieve all tasks not only the ones assigned to the user.

12 November 2021
API v4.70
FIX: Fixed permalinks in Bamboo and Factorial connectors.

FIX: Added created_by to all time offs.

FIX: Updated switch subscription endpoint to automatically delete discounts when switching to an annual plan.

IMPROVEMENT: Updated list time offs endpoint to return no records if the time off app is turned off.

IMPROVEMENT: Added jira option in task management app.

IMPROVEMENT: Updated Intercom cronjob to add more company attributes (hours per month, files, apps, etc).

IMPROVEMENT: Added account status SUBSCRIBED_CHURNED for accounts that have already canceled their subscription.

IMPROVEMENT: Added Sentry logs for BadRequest exceptions.

IMPROVEMENT: Added Sentry logs for JDBC exceptions in update user endpoint.

FIX: Deleted event tags in add, update event and track task endpoints.

FIX: Fixed an issue when a notification for the session user would be created when he added a new comment.

FIX: Fixed an issue in add subscription endpoint to block customers from subscribing when they already have an active subscription.

IMPROVEMENT: Improved send reset password email to disregard unsubscribed settings.

IMPROVEMENT: Fixed Synchronization exceptions in MS and Google calendar cronjobs.

FIX: Fixed multi-account is in get apps in login endpoint. Now, apps are returned for the default team.

IMPROVEMENT: Added host name in response headers.

FIX: Fixed duplicate issues in task user relations.

FIX: Fixed bulk manipulation exceptions in Cookie Cleaner cronjob.

FIX: Fixed stop all schedulers endpoint.

FIX: Fixed list account task with filter tracking to validate that a tracking task has at least one tracking event linked to it in the task user relation..

IMPROVEMENT: Added is_owner and role to Intercom user attributes.

IMPROVEMENT: Added smart project selection for third party projects.

FIX: Fixed pagination issue in Asana connector.

FIX: Fixed update issues in Basecamp cronjobs.

FIX: Fixed Asana connector to retrieve all tasks available to users rather than only tasks that have been assigned to him.

FIX: Fixed delete third party tasks.

FIX: Fixed project relations to third party services. Any given project can only be linked to a third party service at a time.

IMPROVEMENT: Added user role to all connectors that support importing users.

NEW: Added import users endpoints for third party services (Bamboo, Factorial, Asana, Basecamp)

IMPROVEMENT: Added occurrences validation in delete custom field and enum option endpoints.

NEW: Added custom fields in events/billed and events/not_billed endpoints.

NEW: Added appearance parameter in user settings.

IMPROVEMENT: Added quantity_hours and unit_amount_hours in invoice items endpoints.

FIX: Fixed rounding issues in invoice items endpoints.

NEW: Added import projects for third party connectors Asana, Basecamp and Trello.

25 October 2021
API v4.69
FIX: Fixed null description in list time offs endpoint.

FIX: Fixed order in list time offs endpoint.

FIX: Fixed list tasks in Basecamp 2 and Basecamp 3 connectors.

FIX: Fixed list time off events in list events endpoint filtered by account.

IMPROVEMENT: Refactored update user endpoint to use multiple atomic transactions and avoid deadlock issues.

FIX: Fixed timezone issue in Google Calendar integration.

IMPROVEMENT: Added Google support in refresh token endpoint.

IMPROVEMENT: Updated Intercom Cronjob to include new company attributes.

FIX: Fixed an issue in import events endpoint where multiple relations between a task and a user would be created.

IMPROVEMENT: Added support for 'everyone' and unassigned tasks to import projects endpoint.

NEW: Added endpoints to create, retrieve and delete demo data.

NEW: Added backoffice endpoint to fix task user duplicates.

FIX: Fixed cross account issues in import users endpoints in Factorial connector.

IMPROVEMENT: Added web socket messages in turn app on and off endpoints.

IMPROVEMENT: Added web socket messages in update app settings endpoint.

IMPROVEMENT: Added web socket messages in update user endpoint.

05 October 2021
API v4.68
FIX: Fixed cronjob issue in Factorial where deleted timeoffs would not be synced.

FIX: Fixed permalink issues in Factorial Connector.

FIX: Fixed cronjob issue in Factorial where email invites to newly imported users would not be sent.

IMPROVEMENT: Updated timeoff endpoints to set notes to empty string instead of null.

IMPROVEMENT: Added email notification to users when problems occur in Microsoft and Google Calendar integrations.

IMPROVEMENT: Added error status in Microsoft and Google Calendar integrations.

FIX: Fixed timezone issue for Turkish users in Microsoft Calendar integration.

FIX: Fixed permission issues in turn app off endoint where co-workers and project managers would not be able to turn off user apps.

IMPROVEMENT: Updated turn user app off endpoints to automatically delete third party cache data that belongs to the app.

FIX: Fixed email issues in update time off endpoint.

NEW: Added archive projects in batch endpoint.

NEW: Added re-open projects in batch endpoint.

NEW: Added ping endpoints for all cronjobs for monitoring.

NEW: Added signin with Asana endpoints.

IMPROVEMENT: Added user color index for time off events in list events endpoint.

FIX: Fixed pagination issues in Basecamp 2 and 3 Connectors.

27 September 2021
API v4.67
FIX: Fixed hourly rates in list and get event endpoints.

IMPROVEMENT: Refactor list events endpoint to improve performance when selecting long periods of time (4+ years).

IMPROVEMENT: Refactor update and delete task endpoints to avoid deadlock issues when updating / deleting tasks in batch.

NEW: Added Server Timing API in list events endpoint.

IMPROVEMENT: Added web socket messages in time off endpoints.

NEW: Added weekly schedule reminders.

IMPROVEMENT: Updated transaction email scheduler to bypass unsubscribe settings for critical emails like invite and report exports.

IMPROVEMENT: Updated content header for JSON responses to application/json

NEW: Added new endpoint to retrieve users without work schedule.

NEW: Added Basecamp 3 add on.

NEW: Added Basecamp 2 add on.

NEW: Added Asana add on.

NEW: Added Trello add on.

NEW: Added Factorial add on.

NEW: Added task logs (starting Sep, 28).

NEW: Added default system custom fields for tasks (timeline).

IMPROVEMENT: Updated custom fields endpoint to avoid deleting all currently relations to entities.

NEW: Added time off emails.

NEW: Added authenticated accept invite endpoint.

NEW: Added leave workspace endpoint.

14 September 2021
API v4.66
FIX: Fixed connect to Asana when new user IDs would exceed the integer limit in Java.

FIX: Fixed an issue where multiple account apps would be created at login for calendar add ons.

NEW: Added cronjobs for Factorial and Bamboo.

NEW: Added cronjobs for Asana and Trello.

FIX: Fixed issue with permissions and tracking policies.

FIX: Fixed issue in import holidays where all holidays would be marked as half day.

NEW: Added time off events in list events endpoint (with filters Company and User) including time offs and holidays.

FIX: Fixed connection pool issues in API Guard.

NEW: Added import holidays from Factorial.

IMPROVEMENT: Added app constraints validation for time offs.

IMPROVEMENT: Fixed issue in work schedules when using null instead of cero for not-workable days.

IMPROVEMENT: Added time off emails for accepting and rejecting time offs.

IMPROVEMENT: Added websocket messages in time off endpoints.

IMPROVEMENT: Refactored list events query to improve its performance when running reports for a long period of time.

FIX: Fixed timezone issues in Microsoft Calendar.

FIX: Fixed issue at login with multiple relations with account and Microsoft and Google Calendar add-ons.

NEW: Added task logs.

NEW: Added endpoint to return users that have no work schedule assigned to them yet.

FIX: Fixed cross-account issue in list third party users (Bamboo, Asana, Factorial).

NEW: Added Asana and Trello add-ons.

NEW: Added Factorial add-ons.

NEW: Added Beta Testing add-on.

NEW: Added Basecamp 2 and Basecamp 3 Connectors.

20 August 2021
API v4.65
IMPROVEMENT: Added approve time offs by email in add time off endpoint.

IMPROVEMENT: Added project id parameter to cleanse account data endpoint to process a single project.

IMPROVEMENT: Removed UserExceptions from Sentry logs.

IMPROVEMENT: Added account status to distinguish between free and paid add ons.

FIX: Fixed an issue in update policy endpoint when multiple policies would be created for one account.

IMPROVEMENT: Added permalinks to third party time offs.

FIX: Fixes and improvements in Outlook and Google Calendar connectors.

FIX: Fixed sorting issues in list assigned tasks endpoint.

IMPROVEMENT: Added app constraints in time off endpoints when Bamboo or Factorial apps are enabled.

IMPROVEMENT: Added statuses in time off endpoints.

NEW: Added time off events in list events endpoint.

FIX: Fixed an issue in count events endpoint to exclude events from logically deleted projects.

FIX: Fixed an issue in upload files endpoint when the uploaded file has no extension.

IMPROVEMENT: Added flag parameter to remove user avatars in list events endpoint.

FIX: Fixed an issue in add holiday endpoint.

IMPROVEMENT: Added service and service task id in update event endpoint.

FIX: Fixed encoding issue in Trello endpoints.

02 August 2021
API v4.64
FIX: Fixed login issue with integrations when user is archived in an account.

FIX: Fixed cross account issue with tracking policies at login.

NEW: Added BambooConnector.

NEW: Added FactorialConnector.

FIX: Fixed Calendar cronjobs when a calendar y turned off and back on again.

FIX: Fixed multi-account issues in integrations at login.

FIX: Removed new Sentry library and re-implemented the old version.

IMPROVEMENT: Updated list logs endpoint to return log entries in the server's standard timezone.

IMPROVEMENT: Added default filter in project user preferences.

IMPROVEMENT: Added archived filter in list projects for Trello and Asana connectors.

FIX: Fixed an issue in enum options when they could not be enabled.

FIX: Fixed an issue at login when the default team was a canceled account.

NEW: Added endpoint to create separate accounts.

IMPROVEMENT: Added schedulers configuration to automatically start / stop cronjobs on web app start / stop.

21 July 2021
API v4.63
NEW: Added list Trello tasks and projects endpoints.

NEW: Added Asana connector with list projects, tasks, workspaces and users endpoints.

NEW: Added Sentry Performance (testing) and updated library to latest version 5.0.1.

FIX: Fixed import users in import projects and import events endpoints.

IMPROVEMENT: Updated list deleted projects endpoint to sort projects by deletion date.

FIX: Fixed encoding issue in Trello and Asana connectors.

FIX: Added custom fields to list project null endpoint.

IMPROVEMENT: Added custom field limits.

FIX: Fixed ordering of enum options in custom fields.

IMPROVEMENT: Added filter parameter in get project min endpoint.

FIX: Fixed encoding issue in websocket messages.

01 July 2021
API v4.62
NEW: Added CTA (call to action) endpoints.

NEW: Added API Guard to implement request quotas.

NEW: Added Trello and Asana app settings.

NEW: Added app bamboo settings.

FIX: Fixed issues in Microsoft and Google calendar cronjobs.

IMPROVEMENT: Updated list user assigned tasks to sort tasks by closed date.

27 June 2021
API v4.61
FIX: Fixed list user timecards endpoint to show the correct from and to parameters.

FIX: Fixed user permissions in get and list integrations endpoints.

IMPROVEMENT: Updated signin with google and microsoft to force users to always select an account.

NEW: Added retrieve Bamboo users and time off types.

NEW: Added method helper to store untranslated string in the dictionary table.

IMPROVEMENT: Improved error messages at login.

IMPROVEMENT: Improved list closed tasks endpoint to order tasks by close date.

IMPROVEMENT: Added include_timeoffs and include_holidays parameters in list events endpoint

FIX: Removed policy note validation in track task endpoint.

FIX: Fixed add account endpoint.

FIX: Fixed list user timecards when timecard range is biweekly.

18 June 2021
API v4.50
FIX: Fixed list recent tasks endpoint to allow retrieving recent tasks for different users.

FIX: Fixed user log errors (dictionary, bookmarks).

FIX: Fixed enum options order in custom fields.

FIX: Fixed memory leaks in Sentry implementation.

IMPROVEMENT: Added websocket messages when automatically creating tasks and projects in track task and add event endpoints.

NEW: Added export endpoints to create export files and list them.

NEW: Added list third party project, tasks and users (Trello).

NEW: Added delete account endpoint.

NEW: Added delete user endpoint.

FIX: Updated custom fields endpoint to allow creating custom fields with the same name for different object types.

04 June 2021
API v4.46.1
FIX: Fixed list events without scheduled events.

IMPROVEMENT: Added smart task selection in add event endpoint.

NEW: Added filter tasks endpoint.

FIX: Fixed null pointer exception in doc endpoints for free users.

IMPROVEMENT: Updated archive users endpoint to stop running user tasks if any.

NEW: Added holiday endpoints (CRUD).

FIX: Fixed signup with Google to automatically create the account's default schedule.

NEW: Added dictionary endpoints (add and list) for terms translations.

NEW: Added move task lists endpoint to allow moving task lists from one project to another one.

FIX: Fixed project accumulated time calculations when moving tasks in batch from one project to another one.

FIX: Fixed inclusion of custom fields in get project min endpoint.

FIX: Fixed include closed tasks per default in get project min endpoint.

FIX: Fixed list assigned tasks endpoint to exclude tasks that belong to softly deleted projects.

31 May 2021
API v4.46
IMPROVEMENT: Refactor get project endpoint for better performance at /projects/min.

IMPROVEMENT: Added new timecard settings.

FIX: Fixed pagination in list events endpoint when using the user filter.

NEW: Added soft delete for projects.

NEW: Added list user timecards endpoint.

IMPROVEMENT: Added cropping and thumbnail support to download data endpoint.

FIX: Fixed duplication issue in in-app notifications about new task comments.

FIX: Fixed allowed origins verification to better support google services.

IMPROVEMENT: Added support for project docs for free accounts.

IMPROVEMENT: Added custom week days support in add event endpoint.

FIX: Fixed custom fields issue in update project endpoint.

IMPROVEMENT: Removed custom fields support for free accounts.

FIX: Fixed import events endpoint.

NEW: Added move task list endpoint.

IMPROVEMENT: Removed projects and tasks limits for free accounts.

IMPROVEMENT: Updated max users limit on subscribed accounts to 1000.

IMPROVEMENT: Added support for task_name and project_name in add event endpoint.

12 May 2021
API v4.45
FIX: Automatically update list_id when updating a task with project_id.

FIX: Fixed pagination in list events endpoint with user filter.

NEW: Added list project users endpoint at .../projects/:id/users.

NEW: Added download data endpoint at .../files/:id/download_data.

IMPROVEMENT: Added available_storage and used_storage in get account endpoint.

NEW: Added list project task lists endpoint at .../projects/:id/task_lists

IMPROVEMENT: Code clean up and refactor of timecards.

IMPROVEMENT: Added custom file name in download file endpoint.

FIX: Fixed check policies for ghost tasks in add and update event endpoints

IMPROVEMENT: Added Sentry logs when allowed origins verification fails.

NEW: Added custom available storage for paid accounts (30GB / active user).

FIX: Fixed custom fields in list events endpoint with using filter when the task has no hours tracked.

IMPROVEMENT: Added task type in list events endpoint.

IMPROVEMENT: Added allowed origin validation and response_type parameter in signin with google endpoint.

FIX: Fixed duplicated in app notifications in add comment endpoint.

FIX: Fixed import projects endpoint by setting tasks' list_position to cero.

27 April 2021
API v4.44
NEW: Added Auditor cronjob.

IMPROVEMENT: Added sort files by creation date.

IMPROVEMENT: Added task name to file JSON object.

NEW: Added project user preferences.

NEW: Added close task list endpoint.

IMPROVEMENT: Added file name uploaded by user.

NEW: Added shared docs for projects.

NEW: Added new project management emails (assign task, assign projects, delete project, close project, etc.).

FIX: Fixed policy verification on update event.

FIX: Fixed multiple users issue in update work schedule.

IMPROVEMENT: Added sort tasks by list position in get project null endpoint

FIX: Fixed issue in add shared doc endpoint when the linked project is deleted.

NEW: Added new download data endpoint for files.

FIX: Fixed sort subtasks endpoint.

FIX: Fixed update task endpoint to allow changing the task list linked to the task.

FIX: Fixed default events issue where a maximum of 30 default events would be returned. Extended the limit to 60.

FIX: Fixed issue in list assigned tasks endpoint when the user has no tasks assigned to them.

IMPROVEMENT: Added customer_count and service_count parameters in account json object and login.

IMPROVEMENT: Added set and available task custom fields in get project endpoint.

IMPROVEMENT: Updated download file endpoint to verify whether we really need to sign it again or not.

IMPROVEMENT: Added file upload limit for free accounts. 5MB for single uploads, and 100MB total available storage.

24 August 2020
API v4.31.1
FIX: Fixed add system notification issues in task endpoints.

FIX: Added support for cross-account websocket messages, used in track and stop.

18 August 2020
API v4.31
FIX: Fixed GoneException issue in websockets delegate where the user would not receive any message if one failed.

IMPROVEMENT: Improved error handling in add task endpoint when the task name exceeds 500 chars.

NEW: Added websocket message in update event endpoint.

NEW: Added websocket message in delete event endpoint.

IMPROVEMENT: Websocket messages are not delivered if the clientID is the same as the connection ID.

IMPROVEMENT: Added websocket message in system notifications (that are not displayed to the user).

FIX: Fixed issue with moneyMode Addon default status. New accounts now have the addon off by default, old accounts on.

NEW: Added CRUD endpoints for Docs.

NEW: Added CRUD endpoints for Custom Fields.

NEW: Added CRUD endpoints for Enum Options.

IMPROVEMENT: Added event json in websocket message event_deleted.

IMPROVEMENT: Updated tokens cleaner cronjob to delete expired tokens after 3 hours.

FIX: Fixed issue in close ghost task where the notification would fail.

13 August 2020
API v4.30
FIX: Fixed issue in get project endpoint where the project would not be found. Now, returning 404 instead of 500.

IMPROVEMENT: Tasks can now be created without specifying any assignees.

IMPROVEMENT: Added 'board' as supported project view.

NEW: Added has_tasks_with_no_project and has_archived_projects parameters in list projects endpoint's response.

NEW: Added /devices/register endpoint to save user device tokens.

NEW: Added cronjob to clean up deprecated device tokens.

NEW: Added AWS API Gateway SDK (aws-java-sdk-apigatewaymanagementapi-1.11.477).

NEW: Added action hooks to send websocket messages using AWS API Gateway.

IMPROVEMENT: Added release_status filter to list apps endpoint. Now this endpoint only returns apps with GA status by default.

IMPROVEMENT: Added user token validation in /devices/register endpoint to avoid storing the same connection for a user multiple times.

IMPROVEMENT: The update task endpoint now supports removing all current task assignees when passing users=[]

IMPROVEMENT: Updated update and close task endpoints to automatically make ghost tasks visible when the task name, project or assignees change

IMPROVEMENT: Added include_ghost_tasks option in get project null endpoint.

IMPROVEMENT: Added include_ghost_tasks flag in project/:id endpoint.

IMPROVEMENT: Added Hibernate 4 options to explicitly close the session in v5 public endpoints.

IMPROVEMENT: Updated tokens cleaner cronjob to fire every 30 minutes.

IMPROVEMENT: Removed deprecated Invoice and InvoiceItem objects.

NEW: Removed deprecated AWS libraries: aws-java-sdk-1.4.1-sources.jar, aws-java-sdk-1.4.1.jar, aws-java-sdk-apigatewaymanagementapi-1.11.477

NEW: Added new AWS libraries for S3: aws-java-sdk-s3-1.11.826.jar, aws-java-sdk-kms-1.11.826.jar,jmespath-java-1.11.826.jar

NEW: Added new AWS libraries for API Gateway (websockets): aws-java-sdk-apigatewaymanagementapi-1.11.798-SNAPSHOT.jar, aws-java-sdk-core-1.11.798-SNAPSHOT.jar,httpclient-4.5.12.jar,httpcore-4.4.13.jar,joda-time-2.10.6.jar

IMPROVEMENT: Added index parameter in /tasks/share endpoint.

FIX: Fixed an issue in delete project endpoint that caused the object not be moved to the trash.

IMPROVEMENT: Added clientId in all websocket messages.

NEW: Added websocket message in update event endpoint.

NEW: Added websocket message in track / stop task endpoints to notify all account admins that a working on update is required.

NEW: Added Messenger Scheduler to send websocket messages when notification updates are required.

IMPROVEMENT: Improved exception handling in websockets delegate and added Sentry logs.

IMPROVEMENT: Added type validation in add and update task endpoints.

IMPROVEMENT: Added support for '[]' values in users parameter in add and update task endpoints.

FIX: Removed type EVERYONE as default when removing all users in update task endpoint.

FIX: Fixed an issue in update task endpoint when the parameter users in the response would always be empty.

FIX: Refactored data import endpoint to avoid nested connections issues in v5.

FIX: Fixed an issue in update task endpoint where stoping a shared task would sometimes throw a null pointer exception.

FIX: Fixed api version issue in all responses that caused to web app to be automatically refreshed. Now, using always 4.0.

IMPROVEMENT: Improved db exception handling in v5.

19 July 2020
API v4.29
FIX: Added is_project_favorite in /users/:id/tasks/tracking endpoint.

NEW: Added default_view in projects endpoints (add, update, list).

NEW: Added support for subscriptions with multiple products.

IMPROVEMENT: Refactored apps endpoints to support product subscriptions.

24 June 2020
API v4.28.4
FIX: Fixed an issue with legacy tasks in list assigned trackables endpoint.

FIX: Fixed comparison exception in search endpoint.

FIX: Fixed an issue in list events endpoint where some coworkers would see no events.

FIX: Fixed filter issue in list user assigned tasks endpoint.

16 June 2020
API v4.28.3
FIX: Fixed Hibernate issue in get project where a nested exception would occur.

IMPROVEMENT: Added parameters project_is_favorite and project_color to get event and list events (min) endpoints.

IMPROVEMENT: Users with the role co-worker can no longer view time entries from other users.

IMPROVEMENT: The parameters "from" and "to" in list events endpoint are no loner required. When these parameters are null the date of the first and latest account's event will be used to limit the time period.

NEW: Added close Hibernate session at the end of each request.

FIX: Fixed query in list events by task id.

FIX: Added project_is_favorite parameter in list assigned tasks endpoint to replace project_following.

FIX: Added billing parameter in minified project JSON object to replace billing_data.

IMPROVEMENT: List assigned users tasks now sorts results by custom priority (1,2,3,0), project name and task name by default.

FIX: Fixed auto conversion of timecard day hours into seconds.

FIX: Fixed user role issue in list events endpoint. Now admins can view all task's time entries.

04 June 2020
API v4.28.2
FIX: Fixed an issue in get user timecard endpoint where the parameter extra_seconds would always be null.

FIX: Fixed an issue in get user tasks (and assigned_tasks) endpoints where the user role was incorrect.

FIX: Fixed an issue in get task endpoint where the project_is_favorite parameter would always be null.

FIX: Fixed multi-account issue in get task opt endpoint.

FIX: Fixed issue in update task endpoint where the parameter project_is_favorite was always null.

25 May 2020
API v4.28.1
NEW: Added new list user tasks endpoint (.../users/:user_id/assigned_tasks) with optimized database queries.

NEW: Added new list user trackables endpoint (.../users/:user_id/assigned_trackables) with optimized database queries.

IMPROVEMENT: Removed no project object in list user trackables with filter.

IMPROVEMENT: Added minify option in new list user trackables endpoint.

IMPROVEMENT: Added project_favorite parameter in the get task endpoint.

IMPROVEMENT: Added automatic hours to seconds conversions in all timecard endpoints.

FIX: Fixed an issue in update project endoint where the project,customer constraint validation failed.

IMPROVEMENT: Removed assignees parameter from list assigned trackables endpoint to improve query performance.

IMPROVEMENT: Disabled time analytics in Intercom cronjob to analyze performance issues.

18 May 2020
API v4.28
NEW: Added days in add timecard endpoint.

IMPROVEMENT: Added support for MS Teams client in user metadata logs.

NEW: Added billing and input tags to manually created events.

NEW: Added preferred version parameter in user metadata and update user settings endpoint.

FIX: Fixed an issue in list user trackables where searching for customer names would cause errors. Now, customer names are not included in the search.

NEW: Added billing and input tags in track task endpoint.

IMPROVEMENT: Added seconds parameters in timecard and timecard day objects (save timecard day, add timecard).

NEW: Added tag "edited" in update event endpoint.

07 May 2020
API v4.27
NEW: Added timecard entity.

NEW: Added list timecards endpoint.

NEW: Added add timecard endpoint.

NEW: Added get timecard endpoint.

NEW: Added request timecard approval and approve endpoints.

NEW: Added delete timecard endpoint.

IMPROVEMENT: Refactored approve timesheet endpoint.

IMPROVEMENT: Added filter parameter to list user trackables endpoint to filter results by task, project and customer name.

FIX: Refactored update policies to use the parameter avoid_tracking_overtime instead of allow_tracking_overtime.

FIX: Fixed an issue in the get user timecard endpoint where no days would be included (conflicting URL paths).

15 April 2020
API v4.26.3
FIX: Fixed policy issue in track task endpoint (tasks/track).

FIX: Fixed an issue in track task endpoint (tasks/track) where using the tags parameter would failed.

FIX: Fixed an issue in the login endpoint where email strings containing tabs would cause multiple issues.

FIX: Fixed ip address issue in list logs endpoint. Now using the value "Unknown" when the original ip address is missing or localhost.

FIX: Fixed user log in batch invite users endpoint. Now adding a new log for every invited user.

FIX: Fixed user log in archive user users endpoint. Now storing the user's object json correctly.

FIX: Fixed user log in activate user users endpoint. Now storing the user's object json correctly.

FIX: Fixed an issue in update event endpoint where checking overtime policies would fail if the duration parameter is null.

FIX: Removed user permission can_view_others from all endpoints. Now coworkers are never allowed to view other users.

FIX: Added better coworker validations in list events and list events min endpoints.

FIX: Added better coworker validations in get task endpoint.

FIX: Added better coworker validations in get project endpoint.

FIX: Fixed json issues in activate app (/apps/policies/on) and update app policies endpoints.

FIX: Fixed an issue in list events endpoint when an error would occur if not using COMPANY or USER as a filter and include_timeoffs is set to true.

FIX: Fixed tracking overtime policy issue in update event endpoint.

FIX: Fixed an issue in get project null endpoint where users with the role coworker would be able to view other users' time entries.

FIX: Fixed an issue in get task endpoint where users with the role coworker would not able to view a task in a project they have access to.

07 April 2020
API v4.26.2
FIX: Fixed list user trackables endpoint for users with co-worker role.
01 April 2020
API v4.26.1
NEW: Added add users to schedule endpoint at /schedules/id/users/add.

NEW: Added remove users from schedule endpoint at /schedules/id/users/add.

IMPROVEMENT: Added projects_count parameter in login JSON response.

IMPROVEMENT: Disabled email bounce notification sent to account owners.

01 April 2020
API v4.26
NEW: Added new user metadata parameters email_bounce_reason and email_spam.

FIX: Fixed an issue in send invites in batch endpoint where the invite link would not be saved.

FIX: Fixed an issue in the SendGrid webhook endpoint that was causing some bounce notifications not to be saved.

29 March 2020
API v4.25.4
FIX: Fixed an issue in search endpoint where using the character ' in the keyword would throw an error. Now escaping the char correctly.

IMPROVEMENT: Updated Intercom Scheduler to start every day at 12am.

FIX: Fixed tasks count in list projects (min) endpoint. Now excluding all hidden tasks.

24 March 2020
API v4.25.3
FIX: Fixed issue in send invite accepted emails. Now using the correct template id.

IMPROVEMENT: Added account owner in team json.

IMPROVEMENT: Updated accept user invite to improve handling of default teams. Now, if the invited user was the owner of another account, the new account will be set as default.

18 March 2020
API v4.25.2
IMPROVEMENT: Increased the limit of list user trackables from 2000 to 14000.
16 March 2020
API v4.25.1
FIX: Fixed issue in Reminder cronjob's invites verification. Now excluding work schedule reminders that are not used.

IMPROVEMENT: Updated get user endpoint to stop retrieving the account's event data. Now include_stats is false by default.

NEW: Added timeoffs in list events (min) endpoint.

NEW: Added timeoffs in get user timecard endpoint.

IMPROVEMENT: Added role parameter in update user permissions endpoint (/users/update_permissions).

IMPROVEMENT: Added include_tasks flag in list user trackables endpoint. Default: true.

IMPROVEMENT: Added project_id in list user trackables endpoint to filter results by project.

IMPROVEMENT: Deleted old implementation of TimecardsDelegate.

05 March 2020
API v4.25
FIX: Fixed null pointer exception when adding new user metadata (last seen dates).

IMPROVEMENT: Turned logging user logins off by default.

IMPROVEMENT: Added client verification using request header parameter (ClientApp).

IMPROVEMENT: Refactored user metadata parameters for better readibility.

IMPROVEMENT: Added last seen logs to the most used endpoints (track task, list projects, etc).

NEW: Added Asana integration settings endpoint at /apps/asana/update.

IMPROVEMENT: Remove unnecessary parameters from user logs (update user action).

28 February 2020
API v4.24.1
FIX: Removed additional ip addresses from user logs.

FIX: Removed login events from user logs.

IMPROVEMENT: Added minified user logs for improved loading speed.

NEW: Added get user log endpoint at .../logs/:log_id.

IMPROVEMENT: Refactored policy endpoints to be consistent with other apps.

27 February 2020
API v4.24
FIX: Fixed avatar URLs in user groups. Now using the user's compact JSON object.

NEW: Added user metadata to store client logins.

NEW: Added human parameter in login endpoint to identify user logins. Currently set true by default until all clients are updated.

FIX: Fixed rounding issue in user timecard endpoint where the number of extra hours would be incorrect.

FIX: Fixed class cast exception when verifying account policies.

FIX: Fixed null pointer exception in list events endpoint (min) when the id parameter would be missing.

IMPROVEMENT: Updated update event endpoint to allow co-workers who cannot edit time entries to be allowed to edit time entry notes.

IMPROVEMENT: Added hours parameters in seconds format in get user timecard endpoint.

18 February 2020
API v4.23.2
IMPROVEMENT: Added project notes in list events endpoint.
14 February 2020
API v4.23.1
FIX: Fixed null pointer exception in verify user token endpoint.

IMPROVEMENT: Disabled email exceptions in Sentry.

IMPROVEMENT: Added name validation in update user endpoint.

12 February 2020
API v4.23
NEW: Added account policies in add and update event endpoints.

NEW: Added account policies in add, track, sync and update task endpoints.

NEW: Added account policies in add and update project endpoints.

NEW: Added account policies app. Default: off.

NEW: Added apply_to_pms parameter in policy table.

IMPROVEMENT: Updated MAX_LIST_EVENTS_PAGE_SIZE from 14,000 to 25,000.

IMPROVEMENT: Refactored update accumulated time methods to improve time exceeded notifications.

FIX: Fixed null pointer exception in verify user token endpoint.

IMPROVEMENT: Disabled email exceptions in Sentry.

IMPROVEMENT: Added name validation in update user endpoint.

31 January 2020
API v4.22
IMPROVEMENT: Added optional parameters to /schedulers/start endpoint to specify which cronjobs should be started.

IMPROVEMENT: Cleaned up API servlets to improve loading performance.

FIX: Fixed IP address in user logs. Now storing only the user's real IP address.

IMPROVEMENT: Refactored Stripe scheduler to run every 60 minutes instead of hourly.

IMPROVEMENT: Refactored Timecop scheduler to optimize events query for speed.

FIX: Fixed SendGrid's email bounce webhook handler.

IMPROVEMENT: Added default page size limit to list events endpoints.

FIX: Updated MySQL variable interactive_timeout = 300 (seconds, 5 minutes) to automatically kill queries after 5 minutes of inactivity. wait_timeout is also set to 300 seconds.

IMPROVEMENT: Added manager validation in add / update group endpoints. Managers cannot be coworkers.

IMPROVEMENT: Added constraint to add / update user group to allow users to be members of only one user group at a time.

IMPROVEMENT: Updated update user group endpoint with new constraints.

IMPROVEMENT: Added time zone conversion to user logs using the account's default time zone.

FIX: Fixed user logs in project actions.

FIX: Fixed user logs in all signup endpoints.

NEW: Added user logs in delete task comment.

NEW: Added user logs in archive / activate users.

NEW: Added user logs in subscription endpoints.

NEW: Added user logs in user groups endpoints.

NEW: Added user logs in app endpoints.

NEW: Added user logs in data import endpoint.

IMPROVEMENT: Refactored all log action identifiers to store them in the central string dictionary.

IMPROVEMENT: Refactored add user log implementation to use a temporary, separate database session. If an unexpected exception is thrown while trying to save the user log, it will fail silently and don't cause an error in the user request.

IMPROVEMENT: Added AccountLogJson to avoid storing too much data in account logs.

FIX: Fixed issue in signup where two default policies would be created rather than just one.

NEW: Added update account policy log.

NEW: Added support for listing audit logs by account and by object type (without an object_id).

28 January 2020
API v4.22
IMPROVEMENT: Upgraded RDS to db.m3.xlarge.
15 January 2020
API v4.21
NEW: Added join organization endpoint at /account/join.

NEW: Added accept request to join organization endpoint at /account/join/accept.

NEW: Added users hours to get null project endpoint (/projects/0).

NEW: Added merge ghost task implementation to make sure there is always only one ghost task per project and user (currently turned off).

15 January 2020
API v4.20.1
FIX: Refactored get project users hours to avoid performance issues.
12 January 2020
API v4.20
NEW: Updated integrations table to use varchar instead of text for token columns. Added index for access_token column.

NEW: Added refresh access token endpoint at .../integrations/refresh_token/ for all supported integrations.

NEW: Added endpoint to test the Asana integration at .../system/asana/test.

NEW: Added endpoint at /events/count to retrieve the number of account events for a given time period.

IMPROVEMENT: Updated Intercom cronjob to include accounts with an extended trial or free pending confirmation.

IMPROVEMENT: Added user log and Intercom update to accept invite with Google endpoint.

FIX: Added default page and page size parameters in list events endpoints to avoid performance issues.

16 December 2019
API v4.19
FIX: Fixed get user hours in get project endpoint.

NEW: Added employee profile data in new table 'employee'.

NEW: Added new endpoint to update employee data at .../users/:user_id/employee/update.

11 December 2019
API v4.18
NEW: Added invite reminders in invite users in batch endpoint using the standard database session.

NEW: Added more parameters to reminder object.

NEW: Added new reminder cronjob that runs every hour.

IMPROVEMENT: Refactored loading of schedulers properties.

IMPROVEMENT: Added trim email in batch invite users endpoint.

IMPROVEMENT: Added first and last name parameters to the Intercom user.

IMPROVEMENT: Added user log and Sentry logging in accept user invite endpoint. Refactored endpoint for better readability.

FIX: Fixed google analytics parameters in transactional emails.

FIX: Fixed creation of reminders in workplan cronjob by using atomic transactions to store the reminders in the database.

FIX: Updated add event endpoint to reuse hidden tasks when tracking time against a project.

FIX: Updated add event endpoint to update accumulated times when tracking ghost tasks with no project.

28 November 2019
API v4.17.4
FIX: Added rounding to missing hours in reminders.

FIX: Turned off reminders creation in the workplan cronjob due to performance issues.

FIX: Increased the limit of user trackables to 2000.

20 November 2019
API v4.17.3
FIX: Fixed Google Analytics parameters in transactional emails. Added parameters to the dynamic template data.

FIX: Fixed resend email invite's dynamic template data issue where the inviter name would not be included in the JSON payload.

NEW: Added newly invited users to Intercom. Updates status to verified when the user's been verified.

14 November 2019
API v4.17.2
FIX: Fixed issue in accept invite endpoint where the team status would not be updated to VERIFIED.

FIX: Fixed issue in get project endpoint where get project users would result in a null pointer exception when the team is missing.

IMPROVEMENT: Updated signup endpoint to create default schedule with email notifications turned on.

12 November 2019
API v4.17.1
FIX: Fixed shiro configuration error that made the signup endpoint unreachable.

FIX: Added new email templates in resend invite endpoint.

NEW: Added refactored signup with email endpoint using oauth cookies and the verify user token endpoint.

NEW: Added account policy object to our data model and its update policy endpoint.

NEW: Added reminders object to our data model and its endpoints (list, get, open, close).

FIX: Fixed issue in get project endpoint where get project users would result in a null pointer exception when the team is missing.

06 November 2019
API v4.17
IMPROVEMENT: Formated cancellation date in subscription canceled email notification.

IMPROVEMENT: Added clock in / out times validations in add and update schedule endpoints.

NEW: Added new signup with email endpoint.

NEW: Added new verify account with confirmation email and oauth cookie.

28 October 2019
API v4.16.9
NEW: Added Google Analytics to transactional emails.

NEW: Added Google Analytics to SendGrid for tracking clicks and opens.

IMPROVEMENT: Added ghost tasks in list user recent tasks endpoint.

IMPROVEMENT: Added sort by favorites in list user trackables endpoint.

23 October 2019
API v4.16.8
IMPROVEMENT: Removed text from deleted task comments.

FIX: Fixed yet another issue in workplan cronjob that would cause some emails not be saved properly.

IMPROVEMENT: Added email exceptions to Sentry.

IMPROVEMENT: Refactored validations in API mailer and postman cronjobs.

FIX: Fixed several issues in API mailer. No longer using a singleton.

17 October 2019
API v4.16.7
FIX: Refactored list user trackables endpoint to avoid using temporary sql tables. Added 'only_favorites' filter to return only projects marked as favorite by the user.

FIX: Added user stats in list users endpoint when passing the parameter include_stats = true.

IMPROVEMENT: Added accumulated_time in seconds in users_hours JSON in get project endpoint and improved rounding.

IMPROVEMENT: Increased the limit of active tasks in the list user tasks opt endpoint from 1000 to 2000.

IMPROVEMENT: Added task comment object to server response in delete task comment endpoint.

FIX: Fixed an issue in get user timecard endpoint where a day's submitted-by user would always be the session user.

IMPROVEMENT: Added is_workable parameter in the timecard day json object.

13 October 2019
API v4.16.6
FIX: Fixed issue in workplan e-mails where API user preferences being null would result in an error.

NEW: Added new permissions for co-workers 'can_edit_projects" and updated all required endpoints.

FIX: Fixed error in add project endpoint when the user is a coworker who can edit projects.

08 October 2019
API v4.16.5
FIX: Fixed issue with password recovery e-mails.

IMPROVEMENT: Added ghost task id to the users_hours parameter in the project JSON object.

07 October 2019
API v4.16.4
FIX: Fixed issue in method counting the number of failed login attempts. Now using the correct database table.

FIX: Fixed issue with sidebar navigation in new html-based API docs.

IMPROVEMENT: Added coworkers to the project JSON in the get project endpoint.

IMPROVEMENT: Added is_public parameter to the users/:id/tasks/opt and /tasks/:id endpoints.

04 October 2019
API v4.16.3
FIX: Fixed issue in add events endpoint in repeating mode when the repeat_every parameter would be zero.

IMPROVEMENT: If there is an error while trying to get email notification settings, now e-mails are not allowed to be sent by default.

30 September 2019
API v4.16.2
FIX: Refactored API doc in plain html.

FIX: Fixed issue in get project endpoint where the users_hours parameter would not be included in the server response.

28 September 2019
API v4.16.1
FIX: Fixed issue in new save emails method where a missing user's date format preference would throw an error.
25 September 2019
API v4.16
NEW: Added new cronjob (Budgeter) for delivering email notifications when due dates and time estimates are reached (projects and tasks).

IMPROVEMENT: Added parameter formated_due_date to transactional e-mails according to the recipient's settings.

IMPROVEMENT: Added parameter future_events in update event endpoint to update all future instances of the updated event.

FIX: Fixed issue in get user timecard endpoint where 0 scheduled hours would result in shifts marked as extra hours.

FIX: Fixed issue in delete and update event endpoints to include the current object to be deleted / updated.

NEW: Added new security logic to login endpoint to check whether the user has failed too many times within the last minutes.

NEW: Added new failed_login table to store failed login attempts.

FIX: Fixed issue in cancel subscription endpoint.

20 September 2019
API v4.15.2
FIX: Fixed an issue in update user endpoint where the billing data would not be saved for older accounts.

FIX: Fixed issue in Intercom scheduler when updating accounts by individual ids.

NEW: Added users hours in get project endpoint.

IMPROVEMENT: Added worked_hours parameter to list projects (min) endpoint.

IMPROVEMENT: Added sender_name parameter to all transactional e-mails.

FIX: Fixed reply_to parameter in all transactional e-mails.

NEW: Added email notification in assign user to project endpoint.

FIX: Fixed performance issues with new project users hours in legacy list projets endpoint.

11 September 2019
API v4.15.1
FIX: Fixed issues in Timecop cronjob and improved db query.

FIX: Fixed issue in SendGrid's webhook where bounce events would cause an error due to missing support for the new email categories.

IMPROVEMENT: Updated accountant cronjob. Now, users are no longer made admins automatically at the end of the trial period.

IMPROVEMENT: Updated update user endpoint. Free users are no longer allowed to change user roles.

IMPROVEMENT: Added future_events parameter in delete event endpoint to allow users to delete only future events.

IMPROVEMENT: Added repeating parameters in list events (min) endpoint.

NEW: Added delete task comment endpoint at .../tasks/:task_id/comments/delete/:comment_id.

IMPROVEMENT: Added missing parameters for new transactional email templates (project_color, customer, missing_hours, etc).

09 September 2019
API v4.15
IMPROVEMENT: Updated free tasks limit to 100 active tasks.

IMPROVEMENT: Added additional email parameters to the email object in the database (reply_to, manage_link, categories, channel).

IMPROVEMENT: Added day time component to delivery_date and sent_date in email object.

NEW: Refactored Mailer API and all transactional email templates.

22 August 2019
API v4.14.9
NEW: Added new account status FREE_PENDING and refactored instances where the free account status was used.

NEW: Added new endpoint to change the account status (FREE or FREE_PENDING).

IMPROVEMENT: Added new parameter 'uau' (user_avatar_url) to list events (min) endpoint.

IMPROVEMENT: All email notification preferences are now enabled by default for new users.

20 August 2019
API v4.14.9
IMPROVEMENT: Re-added tasks limit (100) for all free accounts.
16 August 2019
API v4.14.8
FIX: Fixed list user tasks with filter archived. Now, co-workers can close tasks that they're assigned to.

IMPROVEMENT: Added project color parameter to search endpoint.

FIX: Added new endpoint to import tasks in batch with JSON input.

02 August 2019
API v4.14.7.1
IMPROVEMENT: Removed tasks limit for all free accounts.
31 July 2019
API v4.14.7
FIX: Added tracking event, loc_worked_hours and accumulated_time_display to JSON response in list user tasks opt endpoint.

FIX: Added tracking event, loc_worked_hours and accumulated_time_display to JSON response in get null project opt endpoint.

IMPROVEMENT: Added "NaN" to parameter validator.

FIX: Fixed issue in subscription cancellation email. Added account and user IDs.

FIX: Fixed an issue in the scheduler superclass where all schedulers would be set in debug mode. Now, using predefined default settings in case something goes wrong while instantiating the schedulers.

FIX: Added new endpoint to return the user avatar for a given user's (secondary) token.

IMPROVEMENT: Added secondary user token parameter in basic user JSON.

IMPROVEMENT: Added secondary user token parameter ("to") in list events (min) endpoint.

IMPROVEMENT: Added note type "tasks limit" and error message in the notes parameter of the server response.

IMPROVEMENT: Added note type "projects limit" and error message in the notes parameter of the server response.

IMPROVEMENT: Added note type "users limit" and error message in the notes parameter of the server response.

IMPROVEMENT: Updated SendGrid webhook to avoid sending user email invalid notifications multiple times.

IMPROVEMENT: Updated close task endpoint. Now allowing coworkers to close their own tasks, regardless of which advanced permissions they have.

NEW: Added new endpoint to securely retrieve a user's temporary token.

10 July 2019
API v4.14.6
FIX: Fixed duplicate project endpoint. Now tasks are shown in the correct order and set as shared tasks. Also, ghost tasks are no longer duplicated.

FIX: Fixed duplicate task issues.

FIX: Swapped endpoint paths for list user tasks endpoints.

IMPROVEMENT: Changed postman transactional cronjob's frequency down to 30 seconds.

16 July 2019
API v4.14.5
FIX: Fixed hours rounding calculations in new get user timecard endpoint.

FIX: Fixed day type in get user timecard endpoint.

FIX: Fixed unsubscribe user in SendGrid in Intercom Unsubscribed webhook.

FIX: Remove string character limit in mark time entries as billed / not billed endpoint.

IMPROVEMENT: Added email confirmation in subscription canceled endpoint.

NEW: Added translations for Spanish and German for all transactional emails that don't use templates.

NEW: Changed billing model for all subscribed accounts using dynamic seats.

FIX: Fixed co-worker permissions in update and delete event endpoints. Now, users with the co-worker role cannot edit other users' time entries.

IMPROVEMENT: Optimized list tasks without project endpoint for speed. Now, loading tasks up to 5x faster.

NEW: Activated new timecards implementation.

FIX: Fixed performance issues in list users endpoint. Disabled include_stats option.

10 July 2019
API v4.14.4
IMPROVEMENT: Improved Stripe (invoice created) webhook.

FIX: Fixed Intercom unsubscribe webhook.

IMPROVEMENT: Updated business logic in get user timecard.

FIX: Fixed billable and non-billable rate issues in list minified events endpoint.

01 July 2019
API v4.14.3
NEW: Added BigQuery integration for storing users in BigQuery data warehouse.

NEW: Added Segment integration for identify, track and group endpoints.

FIX: Fixed email subject issues in SendGrid emails.

FIX: Fixed issue in Intercom's user unsubscribed webhook.

FIX: Fixed email notification in SendGrid bounce webhook.

FIX: New retrieve user tasks implementation using one single query without temporary tables.

FIX: Changed order by in list user tasks query. Now using task id.

19 June 2019
API v4.14.2
FIX: Added order by in add or get task query.

NEW: Now sending work schedule alerts through SendGrid.

18 June 2019
API v4.14.1
IMPROVEMENT: Updated save timecard day enpoint to round hour values.

NEW: Added add SendGrid contacts endpoint for marketing campaigns.

FIX: Fixed issue in Stripe invoice created webhook when new customers signup for the first time.

NEW: Added SendGrid webhook action handling unsubscribe, bounce and spam report events.

NEW: Added Intercom API integration to unsubscribe users.

NEW: Added Intercom webhook action handling unsubscribe events.

NEW: Added unsubscribed parameter to email notification preferences to unsubscrbe users from marketing emails.

NEW: Enabled dynamic seats pricing for all new signups.

FIX: Disabled task opt endpoint again.

11 June 2019
API v4.14
IMPROVEMENT: Added seats and plan_type parameters to account object for improving our billing process.

NEW: Added support for seats and plan types for new pricing in Stripe cronjob.

NEW: Added invoice.created webhook to handle subscription renewals in new pricing model.

IMPROVEMENT: Added seats count and dynamic plan per default in new subscriptions.

IMPROVEMENT: Updated auditor cronjob (user plans) to avoid auditing accounts subscribed to a dynamic plan with seats.

IMPROVEMENT: Project names are no longer unique. You can have several projects with the same name as long as they are assigned to different customers.

NEW: Added SendGrid integration. Now sending transactional emails through SendGrid.

04 June 2019
API v4.13.3
FIX: Fixed issue get timecard endpoint where some dates in 2018 would cause an exception.

FIX: Fixed performance issues in list user tasks optimized and list user trackables endpoints.

FIX: Fixed issue in list user trackables endpoints where public projects would not work correctly.

FIX: Swapped list user tasks endpoint opt with the original one for debugging live in production.

27 May 2019
API v4.13.2
IMPROVEMENT: Added index and default_status parameters to app object.

IMPROVEMENT: Refactored list and update app endpoints

NEW: Added legacy reports app with key legacyReports.

IMPROVEMENT: Added confirmation email in accept invite to notify the admin that a user has accepted the invite.

IMPROVEMENT: Added project notes parameter to list events endpoint.

IMPROVEMENT: Fixed issue in invite users endpoint where account on extended trial could not add more users.

FIX: Fixed issue in update timecard settings where the settings would not be included in the app object.

22 May 2019
API v4.13.1
NEW: Added app timecards settings in list apps and activate app endpoints.

NEW: Added update app settings endpoint at /apps/:app_key/update.

FIX: Fixed query issues in activate / deactivate app endpoints.

IMPROVEMENT: Updated business logic in extra hours calculation (user timecard endpoint) to take the user's working hours into account.

IMPROVEMENT: Refactored activate / deactivate endpoints.

21 May 2019
API v4.13
IMPROVEMENT: Added total_hours parameter in get timecard and save timecard day endpoints.

IMPROVEMENT: Added sort by user name in schedules JSON object.

IMPROVEMENT: Added break hours as timecard entries in generate user timecard endpoint.

17 May 2019
API v4.12
NEW: Added save timecard day endpoint.

IMPROVEMENT: Improved get user timecard endpoint.

IMPROVEMENT: Improved save timecard day endpoint.

NEW: Added timecard day request logs.

13 May 2019
API v4.11.2
FIX: Fixed timecard issues with extra hours in get timecard endpoint.

FIX: Fixed time entry issues in data import tool.

12 May 2019
API v4.11.1
FIX: Fixed issue in activate / deactivate app endpoints where the regex pattern match would fail when adding the account id to the request url.

NEW: Added get timecard endpoint to retrieve a user timecard for a given time period.

IMPROVEMENT: Added daily clock-in and clock-out times for work schedules.

IMPROVEMENT: The data import tool now supports shared tasks.

FIX: Fixed issue in Postman cronjob when re-sending email invites.

FIX: Fixed issue in invite users endpoint where free accounts would not be able to add more than three users.

FIX: Added again last_updated_at, hours_last_week, hours_last_month parameters in Intercom cronjob.

05 May 2019
API v4.11
FIX: Fixed object not found issue in track task endpoint (/tasks/track).

FIX: Fixed loading image issue with SSL and added https to gravatar URLs.

NEW: Added apps model objects and CRUD endpoints.

NEW: Added model objects for timecards and timecard entries and CRUD endpoints.

NEW: Fixed time tracking issues in track task endpoint (tasks/track) where old tasks would collide with shared tasks.

29 April 2019
API v4.10.14
IMPROVEMENT: Added autostop timer logic in sync track endpoint (tasks/sync) to avoid long, inaccurate time entries.

FIX: Added trim and lower case methods to task and project names in track task endpoint (tasks/track?task_name=&project_name=) to avoid creating false duplicates.

IMPROVEMENT: Changed pricing logic in annual plans. Now, when users are archived the customer will get a prorated credit on their upcoming invoice .

FIX: Fixed issue in Postman transactional cronjob where password recovery emails would cause a null pointer exception.

IMPROVEMENT: Refactored smart task selection in track task endpoint (tasks/track) to support shared tasks and project permissions.

24 April 2019
API v4.10.13
IMPROVEMENT: Added limit of maximum emails allowed to send per day by an account to Postman cronjob.

NEW: Added new postman cronjob for delivering only transactional emails.

FIX: Fixed ImageDispatcher by replacing the endpoint logic by Amazon's serverless service.

IMPROVEMENT: Re-added parameters to Intercom such as projects, archived projects, hours_last_week, etc..

17 April 2019
API v4.10.12
IMPROVEMENT: Added user limits in add users in batch endpoint (users/batch_invite).

NEW: Added account trusted parameter in account.metadata.is_template for security reasons.

NEW: Refactored all transactional emails to add them to the email queue instead of sending them instantly.

15 April 2019
API v4.10.11
FIX: Fixed issue in list user trackables where the project color would always be null.

FIX: Fixed issue in list user trackables endpoint where the inlcude_all_projects would not work for users with role admin or project manager.

FIX: Refactored list user tasks optimized and list user trackables endpoints to avoid making unnecessary DB queries, which caused severe performance issues.

NEW: All account users are now set as Admins when accounts are downgraded to the free plan at the end of their free trial.

NEW: Added restart trials cronjob.

IMPROVEMENT: Default work schedules are now created with notify users false during sign up to avoid sending too many emails during onboarding.

08 April 2019
API v4.10.10
FIX: Added limit of 1000 objects to list user tasks optimized.

IMPROVEMENT: Changed name of default avatar images to use new versions.

28 March 2019
API v4.10.9
NEW: Refactored list user tasks endpoint at /users/:user_id/tasks/opt to improve response times.

IMPROVEMENT: Removed all account canceled / suspended and action forbidden exceptions from Sentry.

NEW: Added new merge projects endpoint (/projects/merge/:id).

FIX: Fixed several issues in list user trackables endpoint.

FIX: Fixed several issues in list user tasks optimized endpoint.

FIX: Fixed sort collections across all endpoints where upper case names would be incorrectly sorted.

IMPROVEMENT: Updated Intercom scheduler to include free accounts that have been seen within the last week.

NEW: Added option to update specific accounts in the Intercom scheduler.

14 March 2019
API v4.10.8
FIX: Fixed list user recent tasks endpoint (/users/:user_id/tasks/recent). Now showing tasks tracked today, in the correct order.

FIX: Fixed ArrayIndexOutOfBoundsException in search endpoint (/search) when user names or surnames would be missing.

FIX: Fixed null pointer exceptions in delete endpoints in cases where the requested object could not be found.

IMPROVEMENT: Removed all user exceptions from Sentry.

IMPROVEMENT: Disabled AWS logging.

12 March 2019
API v4.10.7
IMPROVEMENT: Added limit of 500 results to search endpoint.

IMPROVEMENT: Changed limit of list projects (min) endpoint to 5000 results.

FIX: Fixed a mysql query issue in list user trackables endpoint when include_all_projects true.

FIX: Fixed an issue in list trackables endpoint where customer ids and names would not be included in the query response.

08 March 2019
API v4.10.6
IMPROVEMENT: Improved search by adding support for multiple keywords separated by space.

IMPROVEMENT: Removed hidden tasks results in the recent tasks endpoint (/tasks/recent).

IMPROVEMENT: Added more parameters (color, estimated_time, accumulated_time, due_date, customer) to json response in user trackables endpoint.

06 March 2019
API v4.10.5
FIX: Fixed array out of bound exception in new search endpoint.

FIX: Fixed users null issue in new search endpoint, where older tasks with a single task assignee didn't include the user object.

FIX: Added hourly rate calculation in list events endpoint taking the task, project and user rates into account.

FIX: Added task user status in search result JSON.

FIX: Added task user status validation in search endpoint for coworkers. Now, only active assignments are included in the search results.

IMPROVEMENT: Deactivated X-Ray logging.

27 February 2019
API v4.10.4
FIX: Added user_groups, user_hourly_rate and user_hourly_cost parameters to user JSON object in add, get and update event endpoints.

FIX: Updated update user endpoint to block users with role project managers to change their role to admin.

FIX: Fixed an issue in new list projects min endpoint, where some projects would be duplicated on the list.

FIX: Fixed an issue in follow project where project memberships would be incorrectly duplicated.

FIX: Fixed an issue in unfollow project where project memberships would be incorrectly duplicated.

FIX: Added following parameter in list user trackables endpoint.

FIX: Project managers are now not allowed to edit users' hourly_cost and hourly_rate parameters.

IMPROVEMENT: Desktop clients are now using CloudFront using https://api.trackingtime.co

25 February 2019
API v4.10.3
NEW: Added new optimized search endpoint at .../search.

NEW: Added new endpoint /user/trackables to list projects / tasks assigned to the user. Added flag parameter include_all_projects.

IMPROVEMENT: Updated list projects limit to 5000 in the new projects/min endpoint.

22 February 2019
API v4.10.2
NEW: Added new optimized search endpoint at .../search.

FIX: Fixed estimated time format error in list projects min endpoint.

FIX: Added billing data in list projects min endpoint.

19 February 2019
API v4.10.1
IMPROVEMENT: Updated work schedule email template to include manage preferences link. Improved naming consistency across all templates.

FIX: Fixed null pointer exception issue in open and close subtask endpoint when the linked task had no assignees yet.

NEW: Added show_message parameter in user settings to indicate whether we should show new product updates (true) or not (false).

NEW: Added user_hourly_rate and user_hourly_cost parameters to update user endpoint and user json objects.

NEW: Added user_hourly_rate and user_hourly_cost parameters to list events endpoints.

NEW: Added new minified endpoint to list account projects (.../projects/min). Supports public projects.

NEW: Added support for public projects in list projects endpoint (for coworkers).

IMPROVEMENT: Optimized search projects endpoint for speed and added support for public projects.

IMPROVEMENT: Updated default user role from coworker to project manager.

04 February 2019
API v4.10
NEW: Added value parameters to event tags to store key-value custom fields.

NEW: Added save event tag and remove tag from event endpoints.

NEW: Refactored event queries to include event tags.

NEW: Added event tags to json objects.

NEW: Added list tag values endpoint (.../events/tags/values).

NEW: Added tags parameter to track task endpoint.

NEW: Added Sentry integration for better debugging.

NEW: Added public parameter to project object (add, update, get, list).

NEW: Refactored list user tasks to include all public projects for co-workers (at .../users/:id/tasks/pp for testing).

NEW: Refactored get project endpoint to verify whether the project is public or not. Co-workers can access all public projects.

FIX: Fixed null pointer exception issue in delete event endpoint when the event's task was currently being tracked.

11 January 2019
API v4.9.5
NEW: Added list user's recent tasks endpoint at /users/:user_id/tasks/recent.

IMPROVEMENT: Added session user's email address as reply-to email in new task comments.

FIX: Fixed list events by service endpoint.

FIX: Fixed cross account issue in event's user groups.

NEW: Added cors.preflight.maxage = 600 to reduce response times on preflight requests (OPTIONS), both on web app configuration (web.xml) and requests' response context.

NEW: Added new service desk action to update email notification preferences.

NEW: Added bookmark type API_USER_PREFERENCES to store email notification preferences.

NEW: Added service desk action to get the user's email notification preferences (/users/email_prefs).

NEW: Added new email notification preferences to update user endpoint.

NEW: Updated email controller to create / retrieve service desk cookie and verify the user's snooze preferences.

NEW: Updated all transactional emails to include the manage email notification preferences link.

NEW: Updated cronjobs (Postman, Weekly Report, Workplan, Timecop) to use the new transactional email workflow. Added manage_prefs_link to all json metadata objects used in Mandrill templates.

NEW: Added managed emails support in test email endpoint.

NEW: Refactored add task comment endpoint to use single email method instead of an array of emails to implement the new email workflow.

NEW: Added color parameter in import json data endpoint.

NEW: Added support for Czech language in user settings.

NEW: Added users and projects count in team json object.

FIX: Fixed broken accept invite link in re-send invite endpoint.

FIX: Fixed in-app notifications in shared task endpoint (removed unused user setting notification_on_new_task).

FIX: Fixed in-app notifications in close task endpoint.

FIX: Fixed email notifications in close task endpoint when the task is a shared one.

FIX: Fixed issue in switch plan endpoint to avoid switching a customer on an annual plan to another annual plan.

FIX: Fixed user group JSON to include an empty array in the users parameter, rather than null, in case there are no users associated to the group.

IMPROVEMENT: Added verification of schedule's email notification setting in workplan cronjob to avoid retrieving accounts that don't want to alert users of incomplete timesheets.

IMPROVEMENT: Added delivery date in email objects stored by cronjobs (reporter, workplan, timecop) in the email queue table.

07 December 2018
API v4.9.4.1
FIX: Fixed broken accept invite link in invite user emails.
06 December 2018
API v4.9.4
IMPROVEMENT: Updated Stripe API version in all endpoints. Now using 2018-09-24.

IMPROVEMENT: Added email header parameter Reply-To to all Mandrill transactional emails.

FIX: Fixed broken dashboard links in transactional emails.

FIX: Turned off list user logs endpoint for performance reasons.

FIX: Updated add user groups and update user groups endpoints to avoid using action flags add and remove.

IMPROVEMENT: Added sort by name in users parameters in user group JSON.

FIX: Updated add and update user group endpoints to validate that the user is not already assigned to the user group.

NEW: Added new list minified events endpoint to reduce the size of the server response.

NEW: Added is_billed parameter in add event endpoint.

26 November 2018
API v4.9.3.2
FIX: Changed max project list back to 500.
23 November 2018
API v4.9.3.1
FIX: Fixed null pointer exception in list tracking tasks endpoint (/tasks?filter=TRACKING).

FIX: Fixed null pointer exception in get project endpoint.

NEW: Added do_action parameter in login endpoint and new login workflow for extended trials.

NEW: Added note and note_type parameters to all server responses.

NEW: Added test endpoint to stream data to BigQuery's Marketing datasets.

NEW: Added user_groups parameter in list events endpoint and refactored retrieve events query.

IMPROVEMENT: Added parameters trial_ended_at and schedules (count) to Intercom.

12 November 2018
API v4.9.3
NEW: Added preferences parameter to user JSON object.

NEW: Added filter by name and type in list bookmarks endpoint.

NEW: Added restart_trial parameter to account metadata.

NEW: Added BigQuery API delegate and required libraries.

NEW: Added mark events as billed batch action at events/billed.

NEW: Added mark events as not billed batch action at events/not_billed.

NEW: Added delete events batch action at /events/delete.

NEW: Added save bookmarks endpoint (bookmarks/save) to smartly update user preferences.

FIX: Updated localized start and end parameters in event JSON model to use the user's time format setting, instead of the account's.

FIX: Fixed sync with iCal issues.

IMPROVEMENT: Updated active projects limit from 500 to 700.

IMPROVEMENT: Updated timecop cronjob to send only one email alert rather than one every 6 hours.

06 November 2018
API v4.9.2
NEW: Added trial_started_at and trial_ended_at in account metadata.

NEW: Added /account/restart_trial endpoint to allow users to restart their free trial.

NEW: Added back office utility endpoint to generate cookies for restarting the free trial.

NEW: Added verification of accounts with a renewed trail (ON_TRIAL_EXTENDED) in accountant cronjob.

IMPROVEMENT: Refactored Intercom Delegate in accountant and auditor (unpaid subscriptions) cronjobs to always use the latest update method.

FIX: Fixed issue in update schedule where the notify users settings would not be correctly saved.

31 October 2018
API v4.9.1
FIX: Fixed issue in cancel account endpoint where the subscription would be canceled immediately rather than at period end.

NEW: Added project and project_id parameters to light event JSON object.

18 October 2018
API v4.9
FIX: Fixed null pointer exception in import public holidays endpoint when the optional parameter select_holidays was missing.

FIX: Fixed typo in email address (no-relpy) used for transactional emails.

NEW: Added tags implementation.

NEW: Updated Google GSON library to v2.8.5 and refactored StringMap objects, now using LinkedTreeMap and HashMap objects.

IMPROVEMENT: Updated Stripe library to v7.0.0 and refactored coupons, quantities and other deprecated methods.

NEW: Added subscriptions/add_product endpoint to allow customers to subscribe to multiple products.

NEW: Added subscriptions/delete_product endpoint to cancel individual product subscriptions.

IMPROVEMENT: Added invoice_pdf and hosted_invoice_url parameters in Stripe invoice objects to allow users to download better invoices.

IMPROVEMENT: Added last_updated_at parameter in Intercom.

IMPROVEMENT: Updated all Stripe actions to use their latest API.

IMPROVEMENT: Added template parameter in send test email endpoint to test different custom templates.

FIX: Removed admin emails from daily schedule alerts (workplan cronjob).

FIX: Added customer verification in add subscription endpoint. If the account is already linked to a Stripe customer, we don't create a new one now.

FIX: Fixed inconsistency issue in Stripe's subscription deleted webhook where the account's closed_at parameter would be incorrectly overridden. The subscription deleted timestamp is now only stored in the account metadata (SUBSCRIPTION_DELETED_AT).

FIX: Fixed issue in switch plan endpoint where accounts subscribed to an hours plan would be incorrectly switched to a user plan. Now, an error message is returned to the user to get in touch with us.

IMPROVEMENT: Removed Auditor (Cancellations) cronjob from 'start all schedulers' endpoint. This auditor is no longer required since subscriptions are set to cancel at period end.

NEW: Added Google Analytics to API Doc and improved its layout design for better readability.

IMPROVEMENT: Updated delete task list endpoint. Now, tasks linked to the deleted list are no longer automatically deleted if the parameter delete_all is set to false.

IMPROVEMENT: Updated delete project endpoint. Now, tasks linked to the deleted project are no longer automatically deleted if the parameter delete_all is set to false..

IMPROVEMENT: Updated delete task endpoint. Now, events linked the deleted task are no longer automatically deleted if the parameter delete_all is set to false..

IMPROVEMENT: Updated update event endpoint. Now, if no task or project ids are passed over, the accumulated times are still correctly updated.

22 August 2018
API v4.8.1
FIX: Fixed null pointer exception in open subtask endpoint.

NEW: Added new parameters in Intercom delegate for better reporting (subscribed_at, canceled_at, downgraded_at, went_delinquent_at, went_unpaid_at, recovered_at, gsuite, churned, switched_plan_at, deleted_subscription_at).

IMPROVEMENT: Updated update backoffice account endpoint to automatically update the company in Intercom.

FIX: Fixed monthly spend calculation in Intercom to account for the new plans.

IMPROVEMENT: Updated Intercom cronjob to include free accounts last seen within the last 7 days.

IMPROVEMENT: Updated google oauth endpoints to remove debugging logs.

IMPROVEMENT: Added MandrillDelegate to send transactional emails through Mandrill's REST API to avoid encoding and context length issues for weekly reports.

IMPROVEMENT: Refactored Mailer actions to stop using SMTP and send emails through Mandrill's REST API.

IMPROVEMENT: Improved english copy of transactional emails.

FIX: Fixed email and in-app notification issues in add subtask endpoint.

FIX: Fixed email and in-app notification issues in share task endpoint.

FIX: Fixed issue in Stripe's delete subscription webhook where the account would be canceled instead of downgraded to the free plan.

IMPROVEMENT: Refactored Postman and Reporter cronjobs to send emails through Mandrill's REST API.

FIX: Updated open (un-archive) user endpoint to automatically assign the reactivated user to the default work schedule.

NEW: Added daily digest emails for account admins in workplans cronjob. These new schedule alerts inform admins how many hours their employees tracked the day before.

NEW: Added new service desk endpoint (/schedules/admin-digest/unsubscribe) to unsubscribe users from daily digest emails sent to admins.

NEW: Added new service desk endpoint (/schedules/update_hours). This allows users to update the default scheduled hours. The new hours setting will be applied to all weekdays.

NEW: Added send_email endpoint to test different types of emails through Mandrill.

FIX: Fixed an issue in unsubscribe from schedule alerts where the used token would be incorrect.

14 August 2018
API v4.8.1
IMPROVEMENT: Upgraded RDS storage capacity to 30GB.
13 July 2018
API v4.8
NEW: Added time off entity to data model.

NEW: Added CRUD endpoints for managing time offs.

NEW: Added user log support to time off endpoints.

NEW: Added time off API documentation.

IMPROVEMENT: Updated update group endpoint to make it easier to add / remove users to / from groups.

FIX: Fixed issue in update group endpoints to avoid adding the same user multiple times to the same group.

NEW: Added managed_by parameter in user group for specifying who is responsible for the group.

NEW: Added service desk delegate to handle anonymous requests that performs atomic actions like approving a time off request or stopping a timer.

NEW: Added time off request workflow with email notifications and service desk requests.

IMPROVEMENT: Updated add / update group endpoint to allow users to modify the group's manager (manager_id)

NEW: Added import public holidays endpoint with Public Holiday API.

NEW: Added custom timeoff types to data model. Not supported in endpoints yet.

NEW: Added default work schedule at signup.

NEW: Added public unsubscribe from schedule endpoint.

IMPROVEMENT: Updated workplan cronjob to imlement the daily digest workflow.

FIX: Fixed null pointer issue in Stripe webhooks subscription canceled and payment failed.

10 June 2018
API v4.7.1
IMPROVEMENT: Updated cancel account endpoint to automatically cancel the subscription if any.

IMPROVEMENT: Updated google/signin endpoint to allow G Suite users to install the app from the market place and automatically import domain users.

NEW: Added account metadata (subscribed_at, delinquent_at, etc.) updates in all required endpoints.

30 May 2018
API v4.7.0
NEW: Added user identity to data object model.

NEW: Added google/link_account endpoint.

NEW: Added google/unlink_account endpoint.

IMPROVEMENT: Refactored google/signin endpoint to use the google subject id rather than the email as identifier.

NEW: Added identity_provider parameter in user json object.

IMPROVEMENT: Updated auditor cancellations cronjob to start daily at 23:00 instead of 19:00 and to audit accounts canceled the day before.

NEW: Added account metadata to data object model to store additional relevant parameters.

NEW: Added create account meta data in signup.

NEW: Added Google Signin with OAuth2.

IMPROVEMENT: Updated google/signin endpoint to implement the new OAuth2 workflow.

NEW: Added signin token verification endpoint at users/verify_token.

NEW: Added link google account with OAuth 2.

NEW: Added oauth_cookie to data object model and refactored Google oauth endpoints to stop using the user identity object to store temporary signin meta data.

IMPROVEMENT: Refactored Google endpoints to improve code readability.

NEW: Added import Google users endpoint at /google/users/import

FIX: Fixed issue in Google signin where Google users would be allowed to login into canceled or suspended company accounts.

NEW: Added select_users parameter to google user import to specify whether the user would like to select users before inviting them or not.

15 April 2018
API v4.6.2
NEW: Added action parameter to server response in google signin endpoint.

IMPROVEMENT: Updated Google Signin endpoint's path and token.

IMPROVEMENT: Updated Intercom delegate to use the new access token instead of the API key.

4 April 2018
API v4.6.1
FIX: Fixed issue in Stripe scheduler where accounts with updated users would be canceled instead of updated.

NEW: Added logging to Stripe webhooks for better monitoring.

FIX: Fixed tracked_hours parameter in workplan scheduler's emails in multi-account mode.

IMPROVEMENT: Updated archive user endpoint to remove the archived user from any account schedule he's currently associated to.

IMPROVEMENT: Updated invite users endpoint to automatically associate the new users to the account's default schedule.

IMPROVEMENT: Updated Auditor (Cancellations) scheduler to support auditing accounts that should be canceled at the end of the billing period.

IMPROVEMENT: Refactored data cleansing scheduler to process each account in a separate Hibernate session.

IMPROVEMENT: Refactored cronjobs module to use a singleton Quartz Scheduler.

IMPROVEMENT: Updated all schedulers actions to stop a specific cronjob without stopping others.

IMPROVEMENT: Updated Google's java client libraries to support the latest Oauth versions.

FIX: Fixed issue in payment failed webhook where the new account status would be incorrectly set to SUBSCRIBED_UNPAID_UNPAID.

FIX: Fixed issue in subscription deleted endpoint where the account status would remain subscribed.

NEW: Added Google Signin endpoint.

NEW: Added support for signup, login and accept invite workflows with Google Signin.

DELETE: Removed filter RECENT from list tasks endpoints.

FIX: Fixed error column 'is_default' cannot be null in add / update schedule endpoints.

FIX: Fixed error column 'Object not found' in add / update schedule endpoints.

27 February 2018
API v4.6.0
FIX: Fixed accountant scheduler.

NEW: Added cronpanel table in logs database for monitoring cronjob schedules.

NEW: Added endpoint to start/stop all active schedulers in batch.

IMPROVEMENT: Updated restrictions on free plans to 3 active projects, 100 tasks (for accounts with id > 272000).

NEW: Added auditor cancellations cronjob to cronpanel.

NEW: Added account cancellation webhook for Stripe to cancel subscriptions at the end of the current billing period.

NEW: Added new public integrations servlet to handle anonymous incoming requests from connected 3rd party services.

IMPROVEMENT: Updated Intercom delegate to show when a customer churned by canceling his account at the end of the billing period.

IMPROVEMENT: Updated Stripe scheduler to automatically cancel subscriptions at period end.

09 February 2018
API v4.5.0
NEW: Added schedules object in data model.

NEW: Added CRUD endpoints for schedules.

NEW: Added schedule JSON object in user JSON responses.

NEW: Added sort by name asc in list schedules endpoint.

NEW: Added default parameter in schedule model.

IMPROVEMENT: Updated workplan schedule cronjob to work with new schedule implementation.

IMPROVEMENT: Updated transactional emails to improve copy.

IMPROVEMENT: Updated workplan scheduler to avoid retrieving unnecessary accounts that don't have any schedules set up.

IMPROVEMENT: Updated workplan emails. Now using Mandrill templates.

NEW: Added new plans and updated subscription endpoints and Stripe cronjob.

IMPROVEMENT: Updated accountant scheduler to improve performance and error handling.

NEW: Added support for Stripe subscription trials when using the coupon EXT_TRIAL.

15 January 2018
API v4.4.3
NEW: Added end date limit when adding repeating time entries. The end date must now be within the next 365 days.

NEW: Added limit of 1000 results in list projects endpoint with filter archived.

NEW: Added limit of 5000 results in list project ids endpoint.

NEW: Added limit of 1000 results in list account tasks endpoint with any filter.

NEW: Added limit of 5000 entries in data import (json) endpoint.

NEW: Added limit of 300 results in list services endpoint.

NEW: Added limit of 1000 results in list customers endpoint.

IMPROVEMENT: Updated limit of 500 results in list projects endpoint.

DELETE: Deleted large files (20+ mb) uploaded as profile images from S3.

IMPROVEMENT: Updated list notifications endpoint to use native mysql queries.

IMPROVEMENT: Updated get subscription account endpoint to avoid searching the account by token.

FIX: Fixed search when entering a keyword with upper case letters.

NEW: Added new list user tasks endpoint with pagination (.../users/:id/tasks/paginated).

12 December 2017
API v4.4.2
IMPROVEMENT: Updated Stripe handler for failed payment attempts to store the date in which the status of an account is changed to 'SUBSCRIBED_UNPAID'.

NEW: Added auditor delinquents scheduler to automatically verify unpaid subscriptions.

IMPROVEMENT: Updated update card method to try to automatically pay the latest invoice.

FIX: Fixed permission issue in list user tasks endpoints where project tasks assigned to everyone would show up for co-workers who were not given access to the project.

FIX: Fixed search projects issue for coworkers where matching task results would not show up.

07 December 2017
API v4.4.1
NEW: Added tests for new PDF library to be used for reports and invoices.

IMPROVEMENT: Refactored update intercom methods to sync always the same account details across all endpoints where the intercom update is implemented.

FIX: Fixed update stripe subscription in batch users invite.

IMPROVEMENT: Refactored API documentation for integrations.

IMPROVEMENT: Updated Stripe delegate to charge annual customers as soon as they add new users to their account.

IMPROVEMENT: Updated Stripe delegate to remove automatic user proration when archiving users in an annual user plan.

28 November 2017
API v4.4.0
NEW: Added auditor cronjob to verify subscription cancellations. Fixed cancellation errors in Stripe.

NEW: Added user plans verification cronjob.

NEW: Added hour plans verification cronjob.

IMPROVEMENT: Updated update user endpoint to remove name constraint.

NEW: Added add account endpoint.

FIX: Fixed workplan scheduler in now and debug modes.

NEW: Added workplan logs in cronjob.

IMPROVEMENT: Refactored log table to be used in the logs database.

FIX: Fixed update intercom issue with newly added company hours parameter.

IMPROVEMENT: Improved intercom scheduler to update all subscribed and trial accounts.

NEW: Added default schedule for all available cronjobs.

IMPROVEMENT: Updated Intercom scheduler to add support for firing the cronjob once.

NEW: Added following parameters to Intercom: subscribed_at, closed_at, customers, projects, tasks, archived_users

FIX: Fixed hours calculations for custom periods in Intercom cronjob.

NEW: Added public stats endpoints to retrieve total hours, projects and tasks.

NEW: Added new account stats in get account endpoint.

IMPROVEMENT: Updated Intercom cronjob for better performance when syncing subscribed and trial accounts.

NEW: Added cronjob scheduling table in logs database.

03 November 2017
API v4.3.2
NEW: Added users node in task json log.

DELETE: Removed timesheets from API documentation.

NEW: Added medata to event json (created_at, updated_at).

NEW: Added timezone property files.

IMPROVEMENT: Updated add subscription endpoint adding metada and removing status parameter from request.

NEW: Added hours parameter in Intercom.

FIX: Fixed subscription issue where some users with canceled subscriptions could not re-activate their accounts.

FIX: Fixed Stripe cronjob to update plan quantities when customer is subscribed to a annual plan.

IMPROVEMENT: Updated add subscription endpoint to store the subscription date in the metadata object. Removed metadata update in update account endpoint.

05 October 2017
API v4.3.1
NEW: Added sleep time in accountant cronjob to avoid getting blocked by Intercom's API.

IMPROVEMENT: Updated transactional emails in payment failed webhooks.

NEW: Added custom object log jsons to reduce the size of user logs.

IMPROVEMENT: Updated list user logs to remove title and description parameters.

NEW: Added limit of 20 users in invite users endpoint.

NEW: Added email blacklist to block email spammers and updated signup endpoint to verify email domains.

NEW: Added new version of list projects to add limits and improve performance when user requesting projects is a coworker.

NEW: Added batch invite endpoint to avoid batch update errors and better handling of error messages.

25 September 2017
API v4.3.0
NEW: Added core module and refactored standard components for better modularization ahead of separate deployments.

NEW: Added intercom scheduler to update company status to free.

IMPROVEMENT: Updated scheduler log to start using the separate logs database.

IMPROVEMENT: Updated accountant cronjob to improve debugging.

IMPROVEMENT: Updated weekly report cronjob to include free accounts created within the last six weeks.

NEW: Added debugging emails for Leo.

NEW: Added update account plan in backoffice.

NEW: Added customer validation in add subscription. If there is already a customer with the account's subscription reference the request now throws an error.

NEW: Added user name in user logs.

NEW: Added client name in user logs.

FIX: Fixed null pointer exception in cleanse account data endpoint.

NEW: Added include_archived flag in data cleanse method to specify whether to cleanse archived projects and tasks or not.

NEW: Added Intercom scheduler to update the status of companies that finished their trial.

18 September 2017
API v4.2.1
IMPROVEMENT: Updated invoice template to improve format for PDF printing.

NEW: Added all invoices to S3 which have already been issued.

FIX: Fixed unique name constraint on projects.

IMPROVEMENT: Updated data cleanse methods to support shared tasks.

NEW: Added data cleanse method for all account tasks which are not associated to any project.

FIX: Fixed task exceeded notifications.

IMPROVEMENT: Refactored system module to start working on new backoffice endpoints.

NEW: Added scheduler for data cleansing.

NEW: Added search account by id, token or customer email in backoffice.

IMPROVEMENT: Improved update account status in backoffice.

NEW: Added list all account invoices in backoffice.

FIX: Fixed discount issue in print invoice where invoice lines would not display the discounted amount.

FIX: Fixed issue in list and print invoices where free accounts that previously had a subscription would not be able to view their invoices.

IMPROVEMENT: Refactored private system module. Now available at a different URL path.

NEW: Added admin user validation in backoffice endpoints.

IMPROVEMENT: Refactored cronjobs module to remove dependencies with the core API endpoints.

NEW: Added closed_at and closed_by parameters in backoffice accounts endpoints.

07 September 2017
API v4.2.0
NEW: Added list user logs with pagination.

IMPROVEMENT: Updated add project removing unique name constraint.

IMPROVEMENT: Updated add project to allow duplicating projects while keeping task assignments.

FIX: Changed invoice file format to .rtf instead of .txt to avoid encoding issues.

FIX: Fixed issue with coupon parameter in Intercom company updates.

NEW: Added email debugger and cancellation request to Stripe queue in cancel subscription endpoint.

NEW: Added separate database for storing logs. Moved scheduler logs and trash tables to new database.

DELETE: Removed JSON object from user logs in login.

NEW: Added IP address to user logs.

FIX: Fixed IP address issue in trash actions.

NEW: Added action and object filters in list user logs.

IMPROVEMENT: Refactored list user logs endpoint. Now available at .../logs/user/:user_id.

FIX: Fixed quantity and coupon issues in switch to annual plan endpoint.

IMPROVEMENT: Updated endpoint permissions in subscription actions. Now admins can view and print invoices.

14 August 2017
API v4.1.7
NEW: Added user log in login, invite, signup and all add, update and delete actions for all model entities.

FIX: Fixed issue in list projects where accumulated time in task user relation was always cero.

FIX: Fixed print invoice in test mode where receipt number were missing.

FIX: Fixed invoice template to display the correct company name that appears in the bank statements.

IMPROVEMENT: Refactored packages to remove v3 actions for better organization.

NEW: Added Amazon S3 bucket support for permanently storing invoice files.

FIX: Fixed issue in list invoices were there was no upcoming invoice available (canceled accounts).

FIX: Fixed issue with case sensitivity in search projects.

FIX: Fixed issue in search projects when user was a co-worker.

NEW: Added cronjob to automatically end free trials after 14 days.

06 August 2017
API v4.1.6
FIX: Fixed an issue in close subtasks when task assignee was null.

FIX: Fixed date format issue in weekly reports by removing the dates from the subject all together.

NEW: Added custom Quartz properties to automatically stop schedulers when the web app is shutdown.

FIX: Fixed timecop issue in v4.

NEW: Added model classes for user log and implemented standard logging in add and update endpoints.

03 August 2017
API v4.1.5
FIX: Fixed null pointer exception in close to-do when task assigned to everyone.

FIX: Fixed an issue where project tasks assigned to everyone were not listed when retrieving tasks for a coworker.

FIX: Fixed an issue in get user workplan where a project manager would get an error reading that he hasn't has enough permissions.

01 August 2017
API v4.1.4
FIX: Fixed an issue in list tasks where the task user's tracking event was missing in the json response.
30 July 2017
API v4.1.3
FIX: Fixed issues when auto-stopping shared tasks while starting tracking a new shared task.
26 July 2017
API v4.1.2
FIX: Fixed issue in delete time entry for a task assigned to everyone, where there was no task user relation and the error "This user is currently not assigned to this task" was thrown.

FIX: Fixed issue when deleting a time entry for a task which was being tracked (on a different time entry) and the task would incorrectly be auto-stopped.

FIX: Fixed update accumulated time in task user relation in update time entry (single and repeating).

FIX: Fixed update accumulated time in task user relation in delete time entry (single and repeating).

FIX: Fixed update accumulated time in task and project when updating a repeating event.

FIX: Fixed track task when passing project name and task name.

NEW: Changed client version to 4.0 for the public release of shared tasks.

24 July 2017
API v4.1
NEW: Added support for tracking projects by name in tasks/track (used by TT Button).

NEW: Added signup.jsp with redirect to pro subdomain.

NEW: Added limits for free accounts.

NEW: Added API properties to centralize all configuration parameters.

IMPROVEMENT: Refactored Timecop to support shared tasks.

NEW: Added accumulated time to task user relation when sharing an old task.

NEW: Added support for shared tasks in weekly reports.

IMPROVEMENT: Updated trash methods to support shared tasks.

NEW: Added customer data to project object in list user tasks.

FIX: Fixed date localization issue in weekly reports.

NEW: Added endpoint to list all customer invoices.

NEW: Added endpoint to print out an invoice.

NEW: Added bill_to parameter in update account to store billing info to include in invoices.

FIX: Fixed an issue in iCal integration where the subscribe URL was incorrect.

NEW: Added endpoint to switch to another plan.

FIX: Fixed an issue when deleting an event which was currently being tracked by the event's user.

FIX: Fixed an issue in list user tasks when role was co-worker. Now, project tasks assigned to everyone are not included if user has no project access.

FIX: Fixed an issue in list user tracking tasks to avoid showing shared tasks being tracked by other assignees.

FIX: Fixed an issue in add event when task was assigned to everyone and the user was not yet associated to the task. Now the task user relation is correctly added.

FIX: Fixed add task user relation in add event when the task is assigned to everyone and the event's user was not yet associated to the task.

IMPROVEMENT: Refactored update accumulated task to calculate the time based on the task's time entries.

FIX: Fixed update accumulated time in task user relation in add event.

FIX: Fixed update accumulated time in task user relation in add repeating event.

02 July 2017
API v4.0.4
IMPROVEMENT: Updated email subject for new to-do notifications in several languages.

IMPROVEMENT: Refactored cancel account subscription to use the stripe queue.

FIX: Fixed CORS issue with custom error page (down.jsp).

FIX: Fixed URL issue in password recovery email.

NEW: Added uptime folder in API doc to include all uptime reports.

25 June 2017
API v4.0.3
FIX: Fixed confirm email issue. Now using the correct redirect path in v4.

NEW: Added logging in image dispatcher servlet.

NEW: Added now, from and to parameters in weekly report scheduler to fire the cronjob only once.

FIX: Fixed logging in Timecop scheduler.

19 June 2017
API v4.0.2
FIX: Fixed null pointer exception in track task endpoint when task not found.

FIX: Fixed null pointer exception in update event when event not found.

FIX: Fixed null pointer exception in remove user projects when project not found.

16 June 2017
API v4.0.1
IMPROVEMENT: Updated data source configuration to use Tomcat's new native connection pool.

IMPROVEMENT: Disabled unnecessary logging to avoid overhead.

NEW: Added custom tomcat error pages for scheduled maintenance scenarios.

IMPROVEMENT: Customized apache server configuration in new instance (timeouts, max connections, etc.).

FIX: Fixed list user's closed tasks to stop showing tasks cross-account.

FIX: Fixed search projects to avoid case sensitive search.

NEW: Added jsp files for backwards compatibility.

IMPROVEMENT: Updated Shiro configuration to use fallback URLs like in v3.

NEW: Added tomcat.util.http.parser.HttpParser.requestTargetAllow=|{} to catalina.properties to avoid issues with json parameters (TT Button, invite, etc.).

11 June 2017
API v4.0 [Released]
NEW: Added emoji support to database.

NEW: Added more storage in production RDS.

NEW: Added new v4 parameters and tables for shared tasks and user groups.

NEW: Deployed new application infrastructure with Tomcat 8.5 and Amazon Linux machines.

NEW: Added X-ray support in new production machines.

NEW: Added include_all_projects flag in list user tasks to specify whether to show all projects or not.

31 May 2017
API v4.0
FIX: Changed task json objects to always include both parameters "user" and "users" (backwards compatibility).

FIX: Fixed list events issue in Tomcat 8.

FIX: Fixed update old task when assigning it to additional users (backwards compatibility).

FIX: Fixed list events where task was always null.

NEW: Changed API version to v3.

FIX: Fixed response content issue in Tomcat 8.5 when retrieving tasks in some accounts (response content length).

25 May 2017
API v4.0 [Release Candidate]
FIX: Fixed users/:id/tasks?filter=tracking (backwards compatibility).

FIX: Fixed add new task (backwards compatibility).

FIX: Fixed update task (backwards compatibility).

FIX: Fixed list events (backwards compatibility).

FIX: Fixed update event while tracking (backwards compatibility).

FIX: Fixed list company events ascending (backwards compatibility).

FIX: Fixed project accumulated time error (backwards compatibility).

FIX: Fixed add repeating events (backwards compatibility).

FIX: Fixed project null in list events (backwards compatibility).

FIX: Fixed v2/users/:id/tasks (backwards compatibility).

FIX: Fixed v2/tasks/track (backwards compatibility).

FIX: Fixed track task for everyone when no user assigned yet.

FIX: Fixed remove user from task when user is tracking.

FIX: Fixed view task with type everyone when user is co-worker and doesn't have permission.

FIX: Fixed projects/0 to return the correct tasks according to the user's role and permissions.

FIX: Fixed update task when old task becomes shared for everyone (backwards compatibility).

IMPROVEMENT: Updated API documentation.

22 May 2017
API v4.0
FIX: Fixed tasks/track when project id key included and value null.

FIX: Fixed add event with hidden task.

NEW: Added add repeating events with hidden task.

17 May 2017
API v4.0
NEW: Added API version in all server responses.

NEW: Added X-ray support.

FIX: Fixed update task users with type everyone.

FIX: Fixed issue with multiple task user relations when tracking hidden tasks.

NEW: Added notes parameter in tasks/track.

NEW: Added plan parameter in login and user's JSON responses.

FIX: Fixed update event in different scenarios.

FIX: Fixed add event with task and project null. Now using singleton task.

07 May 2017
API v4.0
FIX: Fixed list events in Tomcat 8 temporarily by going back to previous implementation (without DTO).

NEW: Added add and update event with only project id (hidden tasks).

NEW: Added visibility 'HIDDEN' in tasks/tracking.

FIX: Fixed hidden tasks issue in list projects.

NEW: Added task_visibility and task_type in event responses.

FIX: Fixed update and delete event (with task and project null).

IMPROVEMENT: Refactored users/:id/tasks for better performance and to include all projects currently assigned to the session user.

05 May 2017
API v4.0
NEW: Added stop track when removing a user from a task in case he was currently tracking.

IMPROVEMENT: Refactored modify task assignees. Now, the status of the relation is changed (ACTIVE vs. INACTIVE) instead of deleting it.

IMPROVEMENT: Refactored CORS filter. Now using Tomcat 8's native CORS library.

IMPROVEMENT: Updated Tomcat context configuration for Tomcat 8.

NEW: Added track project (hidden task).

NEW: Added track time (without project, hidden task).

NEW: Added visibility and type parameters to task.

23 April 2017
API v4.0
IMPROVEMENT: Refactored update event to support shared tasks and time entries without tasks.

NEW: Added type parameter to tasks (DB, add, update, JSON response).

NEW: Added track task with no user (public tasks).

IMPROVEMENT: Refactored users/tasks to include public tasks in the response.

FIX: Fixed list user tasks and account tasks when task name null.

16 April 2017
API v4.0
FIX: Fixed events/export endpoint after refactoring list events methods.

NEW: Added shared tasks model.

NEW: Added tasks/share to add new tasks and assign them to multiple users.

IMPROVEMENT: Refactored tracking endpoints to support shared tasks.

DELETE: Removed all v4 methods in track, sync and stop task.

NEW: Added user groups to the data model and CRUD endpoints.

IMPROVEMENT: Updated Public API documentation for better reference.

IMPROVEMENT: Refactored add event. Now a task is no longer required to create an event.

NEW: Added user_id parameter in add event.

NEW: Added new endpoint users/:id/tasks/tracking to retrieve all currently running tasks across all user accounts.

13 March 2017
API v3.9.1
FIX: Fixed multi account issue in get user workplan.

IMPROVEMENT: Refactored list events using fetch joins and DTO's for improved performance.

NEW: Added project billing data in /task.

FIX: Fixed icalendar issue (again!).

19 February 2017
API v3.9
FIX: Fixed class cast exception in track task action in v2 (iPhone).

NEW: Added 401 response code (http and internal status) in v4 endpoints when account is canceled or suspended.

NEW: Added trash table for storing deleted projects, tasks lists and tasks as well as its associated events.

IMPROVEMENT: Improved list events action for speed.

FIX: Fixed is billable issue in list events for old accounts.

FIX: Fixed weekly report setting in invite user. Now, true by default.

NEW: Added constraints to schedulers to avoid starting / stopping the same cronjob multiple times.

NEW: Added support for new TLDs in email validation.

12 February 2017
API v3.8.1
FIX: Fixed iCal integration issue for accounts with new tokens.

FIX: Fixed null pointer exception in add event action when duration is missing.

IMPROVEMENT: Refactored track task to allow for stopping tracking tasks across multiple accounts. Newest version works on all API versions.

NEW: Added v4 methods for track, sync and stop tasks cross accounts [On Hold].

IMPROVEMENT: Improved error logging including request URI.

NEW: Added v4 request checks where missing.

NEW: Added include stats flag in list users.

IMPROVEMENT: Refactored list projects to avoid retrieving all tasks from the db to get archived and active counts.

DELETE: Removed json prettify from all server responses.

NEW: Added flags include_subtasks and include_comments in get and list projects.

IMPROVEMENT: Improved cancel subscription action to verify that the subscription status has changed to canceled before updating our account status.

FIX: Fixed issue in get project where session user was a co-worker and had no projects assigned to him.

IMPROVEMENT: Refactored sync task to perform all constraints checks and object validations before processing the request. Newest version works on all API versions.

05 February 2017
API v3.8
NEW: Added new properties in Tomcat datasource configuration to automatically remove and log abandoned connections.

NEW: Added error pages in Tomcat configuration and Shiro (401, 404, 414, 500, 502, 503, 504).

NEW: Added custom logging in all APIActions for user, database and unknown exceptions.

NEW: Added limit of 50 in list notifications action.

NEW: Added list /project/ids action (returns only project id, name and is_archived).

FIX: Fixed error in update user action when trying to update the user role in the current session object (deprecated).

FIX: Fixed error in Image Dispatcher where width or height were missing.

NEW: Added image format validation in file upload action. Now, only png, gif and jpg are supported.

DELETE: Removed free accounts from weekly email report.

NEW: Added v4 api version with http status codes.

IMPROVEMENT: Set wait_timeout = 300 (seconds, 5 min) in MySQL engine to avoid abandoned connections issues.

FIX: Fixed object not found issue in open and delete task.

FIX: Fixed object not found issue in list, open and close subtasks.

FIX: Fixed object not found issue in open / close project.

FIX: Fixed object not found issue in open / close customer.

FIX: Fixed object not found issue in open / close task list.

FIX: Fixed object not found issue in open / close services.

FIX: Fixed object not found issue in v4 for projects, tasks and users.

29 January 2017
API v3.7.3
FIX: Fixed issue in weekly report when using new custom SQL query.

IMPROVEMENT: Changed start of weekly email reports to 05:00am GTM-03.

22 January 2017
API v3.7.2
FIX: Fixed issue in update repeating events where 0 was used instead of null.

FIX: Fixed issue when updating a single instance of a repeating event.

NEW: Added constraint check in add event to avoid creating events with negative duration.

FIX: Fixed query to retrieve active accounts for weekly email reports.

NEW: Added task list in list events endpoint.

08 January 2017
API v3.7.1
FIX: Fixed default billable in add tasks set to null.

NEW: Added debug mode to all scheduled jobs.

NEW: Added system log table.

FIX: Fixed search projects when the user is a co-worker.

NEW: Added better architecture for all cronjobs with properties, abstract jobs and schedulers and a cronjob delegate that handles configuration and error logging.

NEW: Added postman scheduler to send emails in batch.

NEW: Added first version of workplan scheduler for daily checks.

18 December 2016
API v3.7
NEW: Added support for project null or 0 in update task.

NEW: Added sort tasks endpoint.

NEW: Added email log table to improve job schedulers.

IMPROVEMENT: Refactored weekly report to use the email log.

FIX: Fixed add task to include billing data in the server response.

11 December 2016
API v3.6.1
FIX Fixed cronjob start in weekly email report.

IMPROVEMENT Refactored search user tasks to retrieve tasks with no project.

NEW Added cronjob schedulers for sending emails and HTTP requests to external APIs.

IMPROVEMENT Refactored all cronjob processes to use the new atomic schedulers.

IMPROVEMENT Refactored Intercom update to avoid using native threads.

NEW Added user data and project name in search projects response.

FIX Fixed issue in latest update which broke the iPhone app (include stats null).

FIX Fixed issue with billing data in list projects.

FIX Fixed get user workplan.

IMPROVEMENT Refactored update user workplan and removed add workplan endpoint.

FIX Fixed workplan alert scheduler.

04 December 2016
API v3.6
NEW Added export timesheet action with Quickbooks support.

FIX Fixed vulnerability issue in add multiple tasks.

IMPROVEMENT Improved get object for an extra level of security in multi-account endpoints.

IMPROVEMENT Updated set monthly spend in Intercom.

NEW Added new Black Friday coupons.

NEW Added workplan actions.

NEW Added workplan alerts cronjob.

IMPROVEMENT Refactored cancel subscription.

NEW Added search projects action.

20 November 2016
API v3.5.11
NEW Added request timesheet approval action.

NEW Added approve timesheet action.

NEW Added share timesheet action.

NEW Added search tasks action.

FIX Fixed intercom update.

NEW Added update intercom company in add client.

13 November 2016
API v3.5.10
IMPROVEMENT Improved update time entries in timesheet.

NEW Added support for custom entries in add time entries to timesheet.

FIX Fixed bug in add timesheet where project hourly rate was null.

FIX Fixed delete timesheet.

NEW Added request timesheet approval action.

NEW Added approve timesheet action.

NEW Added share timesheet action.

NEW Added export timesheet action.

NEW Added due date in track task.

NEW Added filters in list timesheets.

NEW Added Intercom integration to update company details.

IMPROVEMENT Refactored verify account.

06 November 2016
API v3.5.9
NEW Added automatic user invite and errors response in import JSON data.

FIX Fixed issue in automatic failed payments notifications.

NEW Added metadata in cancel account.

IMPROVEMENT Cleaned up account statuses.

NEW Added add, update and delete timesheet entries.

NEW Added internal notification in update credit card.

NEW Added system endpoint to re-calculate the number of active users in accounts.

23 October 2016
API v3.5.8
NEW Added time entry, timesheet and entry mapping entities to the data model.

NEW Added is_billed in event json objects.

NEW Added is_billed filter parameter in list events.

NEW Added is_billed parameter in update event.

NEW Added CRUD endpoints for timesheets.

NEW Added JSON data import.

NEW Added email notification in update credit card.

NEW Added coupon in signup.

09 October 2016
API v3.5.7
FIX Fixed a bug in subscription actions where the account status was sometimes incorrectly updated.
02 October 2016
API v3.5.6
IMPROVEMENT Refactored event json object to include project billing data as fallback.

IMPROVEMENT Refactored update Stripe subscription. Now using cron job with queue.

NEW Added html templates to payment emails.

25 September 2016
API v3.5.5
IMPROVEMENT Improved transactional emails with html template and cleaner content.

IMPROVEMENT Refactored data import to use the user settings.

DELETE Removed debbuging emails from timecop.

NEW Added export time entries to Quickbooks.

IMPROVEMENT Refactored sync QB customers and service items.

NEW Added event mappings.

IMPROVEMENT Improved password recovery.

18 September 2016
API v3.5.4
NEW Added customer mappings for QuickBooks.

NEW Added user mappings for QuickBooks.

NEW Added service items mappings for QuickBooks.

NEW Added redirect in oauth 1 QuickBooks integration callback.

NEW Added integration already exists verification in add integration. Integration is overwritten in OAuth process.

NEW Added integrations node in account JSON.

NEW Added list, get and delete integration endpoints.

NEW Added mapping tables for customers, users, projects, services and tasks.

IMPROVEMENT Refactored Stripe webhook action. Now sending emails only on the last two failed payment attempts.

FIX Fixed close task. Now removing tracking event if task was running.

IMPROVEMENT Improved close account to automatically stop any running tasks.

IMPROVEMENT Refactored timecop to retrieve only active accounts.

04 September 2016
API v3.5.3
FIX Fixed from and to date filters in weekly report. Now using the current server date as to filter.
28 August 2016
API v3.5.2
NEW Added Log4J logger.

NEW Added Connect to QuickBooks Online via OAuth 1.

NEW Added Signin with Intuit (Single Sign On).

NEW Added list QuickBooks employees.

NEW Added sync QuickBooks employees.

NEW Added list QuickBooks customers.

NEW Added sync QuickBooks customers.

FIX Fixed links and localized texts in time exceeded notifications.

IMPROVEMENT Refactored weekly report query. Now using last login date (within last week) to filter accounts.

21 August 2016
API v3.5.1
IMPROVEMENT Refactored filter to use user settings instead of account settings in autostop timers.

NEW Added estimated time in track task.

NEW Added multi-threading support in weekly report and timecop.

14 August 2016
API v3.5
NEW Added system notifications when task and project estimates are exceeded.

NEW Added CRUD endpoints for bookmarks in v3.

NEW Added invite link in user JSON object.

NEW Added hourly cronjob with email notification for timers that have been running idle for more than 6 hours.

NEW Added support for integrating with 3rd party services via OAuth 1 and OAuth 2.

NEW Added add integration endpoint in v3, supporting Twitter, Google, LinkedIn, Basecamp, Quickbooks, Xero and Debitoor.

31 July 2016
API v3.4.13
NEW Added Tracking filter in list account tasks.

NEW Added system notifications in delete, archive and unarchive project.

FIX Fixed amount issue in payment failed webhook.

DELETE Removed automatic event breakup at midnight in events/update endpoint.

FIX Fixed update event to allow changing tracking events.

NEW Added debugging logs in weekly email reports.

NEW Added new list projects times endpoint.

NEW Added project accumulated time in all task JSON objects.

NEW Added support for Rumanian localization.

24 July 2016
API v3.4.12
FIX Fixed issue in email weekly report where project managers would get the email even though they opted out.

IMPROVEMENT Updated weekly report to avoid sending when no hours tracked last week.

DELETE Removed unique constraint in task list names.

NEW Added automatic email notifications when invoice payments fail.

NEW Added account suspension when a payment fails for the third time in a row.

DELETE Removed verification email in account registration.

DELETE Removed automatic event breakup at midnight in tasks/sync endpoint.

IMPROVEMENT Updated default user avatars.

13 July 2016
API v3.4.11
NEW Added support for updating JSON metadata in tasks/track endpoint.

NEW Added add new card endpoint to create a new card in Stripe.

FIX Fixed link in re-send invite where the account id was used instead of the token.

25 June 2016
API v3.4.10
FIX Fixed bug in verify account where the account status was incorrectly set to on_trial.

FIX Fixed bug in resend invite where the link send was incorrectly formatted.

NEW Added Mandrill subaccount support in weekly email reports.

11 June 2016
API v3.4.9
NEW Changed email service back to Mandrill.

NEW Added weekly email report.

02 June 2016
API v3.4.8
NEW Added list and get system accounts.
21 May 2016
API v3.4.7
FIX Fixed list users for coworkers with 'can view other users' option unselected.

FIX Fixed subscription quantity issue with hour plans.

30 April 2016
API v3.4.6
NEW Changed transactional email provider. Now using Sparkpost.
17 April 2016
API v3.4.5
NEW Added due date, estimated time and archived parameters to list events endpoint.
13 April 2016
API v3.4.4
FIX Fixed set account last login date in login.

FIX Hot-fixed co-worker advanced permissions and image dispatcher.

10 April 2016
API v3.4.3
FIX Fixed project null issue in close task.

FIX Fixed issue in list events by service.

NEW Added upload account logo.

FIX Fixed sort tasks collection when task name null.

FIX Fixed issue in update event where end date null.

IMPROVEMENT Improved list projects to avoid pre-loading of account events when stats not required.

IMPROVEMENT Refactored list projects to include worked_hours when include_stats false.

NEW Added user status verification in get invite.

NEW Added new invite users action in v3.

IMPROVEMENT Improved localizable strings in all model objects.

FIX Fixed file upload in public API.

FIX Fixed email localizations for Polish.

IMPROVEMENT Refactored add task endpoint to set billable by default.

DELETE Removed user token from standard json responses.

NEW Added account status in team json response.

FIX Fixed issue in list events for co-workers who belong to multiple accounts.

IMPROVEMENT Refactored tracking verification to validate that tracking_event is not null.

IMPROVEMENT Updated signup. Now account status is set to ON_TRIAL.

NEW Added account login inactive verification in login.

NEW Added null verification for plans.

FIX Fixed data import bugs.

IMPROVEMENT Refactored email links.

FIX Fixed login where account status canceled or suspended.

NEW Added v3 support in verify account.

FIX Fixed set account last login date in login.

13 March 2016
API v3.4.2
FIX Fixed list events with pagination. Now allowing to specify the order of the results.

IMPROVEMENT Refactored list events for services, customers, projects, tasks and users.

IMPROVEMENT Updated update repeating events to remove group id when only editing.

NEW Added 404 status code in all get endpoints.

NEW Added permissions in user objects and fixed update permissions.

FIX Fixed update json metadata in users with no associated metadata.

IMPROVEMENT Improved user permissions in users endpoints. Now only admins can invite, close and re-open users.

NEW Added user account validation in update user.

NEW Added update notification and autostop timer preferences in user settings.

NEW Added billable and hourly rate parameters in list events endpoint.

FIX Fixed update email_on_comment_updates in user settings.

IMPROVEMENT Updated add comment to notifiy all requested users regardless whether the user to be notified is the session user or not.

IMPROVEMENT Updated add subtask to notifiy all requested users.

NEW Added notifications to all account active users when a new project is added.

10 March 2016
API v3.4.1
IMPROVEMENT Improved system notifications.

NEW Added company logo for exporting reports.

NEW Added update customer card endpoint.

NEW Added get, cancel and update subscriptions.

NEW Added custom v3 notifications.

NEW Added 404 status code in all get endpoints.

NEW Added advanced user permissions with status code 403.

FIX Fixed update customer.

05 March 2016
API v3.4
NEW Added new API v3.

NEW Added duplicate project and project templates.

NEW Added Websockets support.

NEW Added support for Portuguese.

NEW Added support for Mandrill templates in transactional emails.

NEW Added assign / remove users to / from projects.

NEW Added project color.

NEW Added json metadata in all objects.

NEW Added weekly email reports.

NEW Added assign / remove users to / from projects.

NEW Added incoming and outgoing webhooks.

NEW Added Slack and HipChat integrations.

NEW Added CRUD methods for managing account webhooks.

NEW Added json metadata support for accounts, projects, users and tasks.

NEW Added csv data import.

NEW Added Gravatar integration and default profile images.

IMPROVEMENT Improved password hashing, now using 500,000 hash iterations.

NEW Added pagination in list events.

NEW Added json metadata support for accounts, projects, users and tasks.

28 November 2015
API v3.3
NEW Added multi-account support.

NEW Added user settings.

NEW Added polish localization.

NEW Added CORS filter to support cross domain XHR requests.

NEW Added support for Tomcat 8.

NEW Added Stripe Integration.

FIX Fixed task comments time stamps.

NEW Added metadata in event updates.

FIX Fixed break up event at midnight.

NEW Added dynamic task selection in tasks tracking.

NEW Added recent and tracking filters in the list tasks endpoint.

NEW Added task priority.

NEW Added event notes.

NEW Added open/close customers.

NEW Added open/close services.

IMPROVEMENT Updated character limits for database strings.

09 June 2015
API v3.2
IMPROVEMENT Refactored localization with added support for Italian and Korean languages.

FIX Fixed recover password email for german localization.

FIX Fixed email and password enconding issues at login.

FIX Fixed move tasks between projects.

IMPROVEMENT Improved email validation in invite users.

NEW Added timezone support for tracking endpoints.

20 Dec 2014
API v3.1
NEW Implemented Sign in with Google+, Twitter and LinkedIn via OAuth 1.0a and OAuth2.

NEW Added iCalendar feed for user tasks with authentication token.

IMPROVEMENT The user token now is regenerated when he changes his password.

NEW Implemented BasicHttpAuthentication with token.

18 Oct 2014
API v3.0
IMPROVEMENT Improved CSV export.

NEW Added restrictions to user passwords.

05 Oct 2014
API v2.9
NEW Added Public API documentation.

NEW Added User Analytics Report.

FIX Fixed UTF-8 issue.

FIX Fixed export CSV in public API.

14 Aug 2014
API v2.8
FIX Fixed close orphan task.

FIX Fixed spanish emails encoding.

02 Aug 2014
API v2.7
NEW Added worked hours stats in get user endpoint.
25 Jul 2014
API v2.6
NEW Added localized messages in login and signup endpoints.

IMPROVEMENT Refactored track / stop endpoints paths.

NEW Added HTTP header client restriction and account status validations on requests to the public API.

21 Jun 2014
API v2.5
NEW Added data cleansing service.

IMPROVEMENT Improved system copy.

01 Jun 2014
API v2.4
NEW Added HTTP basic auth support for public API.

IMPROVEMENT Improved system copy.

NEW Added team relation for multi-account implementation.

02 Abr 2014
API v2.3
NEW Added orphan tasks.

IMPROVEMENT Improved system copy.

NEW Added move tasks to other projects.

14 Sep 2013
API v2.2
NEW Added login with token.

IMPROVEMENT Updated on-boarding experience.

NEW Added account analytics actions.

1 Aug 2013
API v2.1
FIX Fixed bug in update and track event causing "start date is after end date" exceptions.
7 Jul 2013
API v2.0
IMPROVEMENT Refactored dictionaries to use XML.

IMPROVEMENT Refactored transactional emails to use HTML.

FIX Bug fixes and improvements.

1 Jul 2013
API v1.3.5
FIX Bug fixes and improvements.
20 May 2013
API v1.3.4
FIX Bug fixes and improvements.
10 May 2013
API v1.3.3
FIX Bug fixes and improvements.
21 May 2013
API v1.3.2
FIX Bug fixes.

NEW Added read multiple notifications.

NEW Added Spanish localization.

13 May 2013
API v1.3.1
IMPROVEMENT Nice copy and translations everywhere (EN, DE: 99%, ES: 50%).

NEW Added email and system notifications on re-open task / subtask, archive / re-open project.

FIX Fixed bugs in on-boarding and login.

IMPROVEMENT On close task, all its subtasks are now automatically closed.

NEW Added string length constraints in names, comments, etc.

IMPROVEMENT Updated delete action to make it account safe.

IMPROVEMENT Updated delete and archive user so that these actions can not apply to the account owner.

NEW Added restriction to update owner, so that only admin users can be selected.

IMPROVEMENT Updated delete tasks, so that now all objects associated to it are deleted.

FIX Fixed list events, so that now only events with duration > 0 are retrieved.

FIX Fixed #119. Now sign up on invite does not verify the user if the sign up request throws an exception.

FIX Closed #117. Added resend verification email action at /resend_verification?email=....

FIX Closed #118. Added resend invite action at /users/resend_invite?email=....

21 Apr 2013
API v1.3
FIX Bug fixes.

NEW Added constraints to track and update event.

IMPROVEMENT Optimized hours stats query.

13 Apr 2013
API v1.2.9
FIX Bug fixes.

NEW Added image dispatcher.

NEW Added file uploads on Amazon S3.

NEW Added status code 402 for account suspended .

10 Apr 2013
API v1.2.8
FIX Bug fixes.

NEW Added password recovery.

NEW Added more data in list projects / users / customers.

NEW Added status code 402 for account suspended.

30 Mar 2013
API v1.2.7.2
FIX Bug fixes.

NEW Added notifications on close task / subtasks.

NEW Added list notifications by project.

NEW Added stats in list customers.

NEW Added billable and non-billable hours in projects.

24 Mar 2013
API v1.2.7.1
FIX Bug fixes.

NEW Added new tracking methods.

22 Mar 2013
API v1.2.7
NEW Added notifications on new task, new comment and new to-do.

NEW Added dashboard widget "Team Hours".

NEW Added dashboard widget "Company Hours".

IMPROVEMENT Improved time sheet results.

NEW Added cancel account.

NEW Added account stats and payment history.

19 Mar 2013
API v1.2.6
IMPROVEMENT Updated track and update event. Now calculating duration on server.

NEW Started adding notifications.

15 Mar 2013
API v1.2.5.1
IMPROVEMENT Updated invite people with user verification.

FIX Bug fixes.

11 Mar 2013
API v1.2.5
NEW Added debugger servlet to log errors via email.

NEW Added invite people with user verification.

NEW Added email notification in add subtask.

FIX Fixed sign up and other issues.

04 Mar 2013
API v1.2.4
NEW Migrated DB to start live testing.

FIX Several bug fixes.

FIX Now blocking get requests on objects from other accounts.

NEW Added first iteration of invite users at /users/invite.

NEW Added new account status "first run" on sign up.

18 Feb 2013
API v1.2.3
NEW Added export events action (csv) at events/export.

NEW Added list project users action at /projects/:id/users.

NEW Added email notification in add task comment action.

FIX Fixed track event.

FIX Fixed verify account.

12 Feb 2013
API v1.2.2
NEW Added billing data to get / list / add / update projects, users and tasks.

NEW Added user role verification to get / list projects, users and tasks.

NEW Added freelance parameter to users.

10 Feb 2013
API v1.2.1
IMPROVEMENT Updated localization for dashboard.

NEW Added end_date to get tasks.

NEW Added metadata to update actions.

NEW Added metadata to open/close actions.

IMPROVEMENT Refactored open / close / follow / unfollow actions to return the json object.

IMPROVEMENT Cleaned-up API documentation.

02 Feb 2013
API v1.2.0
NEW Added FastSpring integration (activate subscription).

FIX Fixed update customer contacts.

NEW Added update account action.

NEW Added country and currency codes with locales.

NEW Added account status filter on every action.

NEW Added get FastSpring subscription action.

NEW Added customer localization on API.

28 Jan 2013
API v1.1.9
FIX Fixed customer worked hours.

FIX Fixed delete customer contact.

FIX Fixed signup when no redirect_to is given.

20 Jan 2013
API v1.1.8
NEW Added track event actions at /events/track/:id.

NEW Added update event action at /events/update/:id.

NEW Added sign-up action at /signup.

NEW Added verify action at /verify/[TOKEN].

IMPROVEMENT Updated login action to contemplate different account statuses.

IMPROVEMENT Refactored start,end to from,to in list events action.

NEW Added Signup page in API documentation.

08 Jan 2013
API v1.1.7
FIX Fixed ADD events.

NEW Added filter parameter to GET user tasks.

30 Dec 2012
API v1.1.6
FIX Bug fixes.

NEW Added default sorts for all LIST actions.

NEW Added SORT subtasks.

NEW Added complementary data to event json.

NEW Added default sorts for all LIST actions.

NEW Added chart library and timeline chart.

18 Dec 2012
API v1.1.5
NEW Added GET user tasks grouped by projects at v2/users/:id/tasks.

FIX Fixed #18.

15 Dec 2012
API v1.1.4
FIX Fixed worked_hours, now returning correct values.

IMPROVEMENT Updated ADD and UPDATE events to implement accumulated time calculations (break up events at 00:00 am not implemented yet).

14 Dec 2012
API v1.1.3
FIX Bug fixes.
11 Dec 2012
API v1.1.2
FIX Fixed mail sending bug. Now using Mandril for transactional email.

NEW Added SORT task lists (inside projects).

NEW Added SORT task list (tasks inside task lists).

NEW Added account settings in GET account and documentation explaining the parameters.

NEW Added optional email notification in ADD task.

NEW Added File Uploader at v2/files/add.

FIX Fixed issue #3.

FIX Fixed FOLLOW / UNFOLLOW projects.

FIX Fixed update task_lists.

IMPROVEMENT Updated ADD and UPDATE events. Parameter start no longer required, support for dynamic parameters in UPDATE.

NEW Added LIST events filtered by service, customer, project, user, skill and task.

07 Dec 2012
API v1.1.1
FIX Fixed session closed bug.

NEW Added ADD, GET, DELETE and LIST files actions.

NEW Added Filter by FOLLOWING in LIST projects.

IMPROVEMENT Refactored JSON responses to avoid overhead.

FIX Disabled worked hours stats for debugging.

04 Dec 2012
API v1.0
IMPROVEMENT Updated UPDATE actions to implement new dynamic parameter policy.

NEW Added customer contacts actions.

NEW Added multi-language support for EN, DE and ES.

NEW Added meta data to JSON responses and ADD actions.

IMPROVEMENT Updated UPDATE and ADD project actions to allow creation of customer and services on the fly.

IMPROVEMENT Updated UPDATE and ADD task actions to allow creation of skills on the fly.

NEW Added FOLLOW project action.

NEW Added stats to LIST customers action.

20 Nov 2012
Build v0.9
NEW Added OPEN / CLOSE actions for users, tasks, task lists and subtasks.

FIX Fixed several bugs.

13 Nov 2012
Build v0.8
NEW Added OPEN / CLOSE projects.

NEW Added LIST subtasks.

NEW Added LIST comments.

FIX Fixed several bugs.

05 Nov 2012
Build v0.7
NEW Added filters in GET actions (ALL, ARCHIVED, ACTIVE).

NEW Added includes in GET projects.

30 Oct 2012
Build v0.6
FIX Fixed ADD and UPDATE users.
24 Oct 2012
Build v0.5
NEW Added ADD and UPDATE tasks with task list parameters.

IMPROVEMENT Updated GET events to include associated users.

FIX Set debugging mode off to include error messages in JSON.

22 Oct 2012
Build v0.4
FIX Fixed GET services and skills. Now the filter works properly.

NEW Added parameter 'redirect_to' in login action.

NEW Added reference site to documentation.

NEW Added authentication error in standard JSON response.

NEW Added CRUD-Methods for task_lists.

16 Oct 2012
Build v0.3
NEW Added JSONP functionality to the API.

NEW Added nested object skill to GET tasks response.

NEW Added GET account.

NEW Added Git repository for API.

09 Oct 2012
Build v0.2
NEW Added project actions add, update, delete.

NEW Added nested response objects in GET projects (tasks, customer, service).

IMPROVEMENT Updated API documentation layout and moved to subfolder /doc.

NEW Added code prettyfier to API documentation.

NEW Added doc pages for tasks, users and events.

FIX Fixed get actions in tasks and events.

05 Oct 2012
Build v0.1
NEW Initial deployment.