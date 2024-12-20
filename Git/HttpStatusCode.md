# HTTP Status Codes

This document provides a summary of common HTTP status codes, grouped by their categories.

## 2xx Success Codes

These status codes indicate that the request was successfully processed by the server.

| **Status Code** | **Meaning**                                          | **Usage**                                             | **Example**                                                                 |
|-----------------|------------------------------------------------------|-------------------------------------------------------|-----------------------------------------------------------------------------|
| **200 OK**      | The request has succeeded.                           | The server successfully processed the request.        | A GET request returns the requested data.                                   |
| **201 Created** | A new resource has been created.                     | Returned after a POST request creates a new resource. | A new user is registered, and the server returns the user's details.       |
| **204 No Content** | No content to send in the response.                | The request was successful, but no content is returned. | A DELETE request successfully removes a resource.                           |

## 3xx Redirection Codes

These status codes indicate that the client needs to take additional action to complete the request.

| **Status Code** | **Meaning**                                          | **Usage**                                             | **Example**                                                                 |
|-----------------|------------------------------------------------------|-------------------------------------------------------|-----------------------------------------------------------------------------|
| **301 Moved Permanently** | Resource has been permanently moved to a new URL. | The resource has permanently moved; update the URL.    | A website changes its URL structure, and the old URL redirects to the new one. |
| **302 Found**    | Resource is temporarily located at a different URL. | The resource is temporarily located at another URL.   | Redirecting users to a maintenance page.                                    |

## 4xx Client Error Codes

These status codes indicate that the client has made an error in the request.

| **Status Code** | **Meaning**                                          | **Usage**                                             | **Example**                                                                 |
|-----------------|------------------------------------------------------|-------------------------------------------------------|-----------------------------------------------------------------------------|
| **400 Bad Request** | The server cannot process the request due to a client error. | Invalid syntax or parameters in the request.           | Malformed JSON in a POST request.                                           |
| **401 Unauthorized** | Authentication is required and has failed or not provided. | User authentication is required to access the resource. | User attempts to access a protected API without providing valid credentials. |
| **403 Forbidden** | The server refuses to authorize the request.       | The server denies the request regardless of authentication. | User tries to access an admin panel without permission.                    |
| **404 Not Found** | The requested resource could not be found.          | The resource does not exist or the URL is incorrect.   | Trying to access a non-existent page.                                       |

## 5xx Server Error Codes

These status codes indicate that the server has encountered an error while processing the request.

| **Status Code** | **Meaning**                                          | **Usage**                                             | **Example**                                                                 |
|-----------------|------------------------------------------------------|-------------------------------------------------------|-----------------------------------------------------------------------------|
| **500 Internal Server Error** | A generic error indicating an unexpected server issue. | The server encountered an unexpected error.           | A bug in the server code triggers an internal error.                        |
| **502 Bad Gateway** | The server received an invalid response from the upstream server. | Error occurs while acting as a gateway or proxy.      | Proxy server receives an invalid response from the upstream server.         |
