# Building an Express.js API with Zoom Server-to-Server OAuth Authentication

When it comes to building applications on the Zoom platform, Zoom provides a variety of options to best suite your application's needs. 

![Screen Shot 2022-09-20 at 1 20 22 PM](https://user-images.githubusercontent.com/81645097/191358210-fc012523-2bb0-490e-a090-c736b47ee556.png)

If you take a second to glance over the various app types listed above, you'll see that we are deprecating **JWT** apps in favor of the newly added **Server-to-Server OAuth** app type. This deprecation will take place **June, 2023**. If you're interested to read more about why we're doing this, please see our [JWT Deprecation FAQ](https://marketplace.zoom.us/docs/guides/build/jwt-app/jwt-faq/).

To assist Zoom users with this migration, we will create an Express.js API from scratch and integrate it with Zoom via Server-to-Server OAuth. Let's get started!

## Prerequisites

* Node (https://nodejs.org/en/download/)
* Docker Desktop (https://www.docker.com/products/docker-desktop/)
* Zoom Server-to-Server app credentials (https://marketplace.zoom.us/)
  * **Develop** -> **Build App** -> **Server-to-Server OAuth** create
  * Once created, follow the on-screen directions and add a little bit of information about your app
  * **Scopes**
    * For the purposes of this application, we will add the following scopes:
      * `View and manage all user meetings /meeting:write:admin`
      * `View users information and manage users /user:write:admin`
      * `View and manage all user Webinars /webinar:write:admin`
  * With these scopes selected, navigate back to **App Credentials** and you should see the following pieces of information
    * `Account ID`
    * `Client ID`
    * `Client secret`
  * Keep these handy as we will need them shortly!
  
## Getting Started


    
