# Debug, dockerize, and deploy a Node.js app

## Hotel sentiment analysis front-end
This repo contains a simple Node.js Express web app for viewing sentiment analysis for a fictional hotel chain.

As a hotel manager, you can see the various hotels in your hotel chain on a map (powered by Bing Maps API), and click on them to view customer sentiment for each hotel. Unfortunately, there's something wrong!

In this demo, you'll use **Visual Studio Code** to run and debug the app locally. Once you fix the issue, you'll convert the app to run in a Docker container, then deploy it to the cloud!

## Setup

This section will walk through using the Azure Portal and Visual Studio Code to create all of the local and remote resources you'd need to debug the demo locally, and then deploy the demo to Azure. 

> ***Note: As part of this lab, you'll receive information on how to create and use an Azure Subscription with an Azure Pass.***

### Azure setup
First we'll create your remote resources. Once you've logged in to Azure and have navigated to the portal, create the following resources:

1. Create a Bing Maps API for Enterprise resource in the Azure Portal by clicking the **New** button, then searching for `Bing` and selecting the **Bing Maps API for Enterprise** option. 

    ![Search for Bing](docs/media/01-bing.png)

1. The free tier for public web sites should be appropriate for getting started. 

    ![Selecting the free tier of service](docs/media/02-bing.png)

1. Create a new Azure Container Registry resource by clicking the **New** button in the Azure Portal, then searching for `Registry.` Any of the available SKU options are appropriate. Create the resource in the same resource group as the Bing Maps API for Enterprise resource. 

    > *Note: Use the same resource group when creating the rest of the resources in this lab.*

    ![Creating the Azure Container Registry](docs/media/03-acr.png)

1. Once the Azure Container Registry instance has been created, enable the Admin user as shown in the screen shot below. 

    ![Enabling Admin user](docs/media/04-keys.png)

1. Create a new Basic Linux App Service Plan in the same region and resource group as the Azure Container Registry and Bing Maps API for Enterprise resources. Use the **B1 Basic** SKU.

    ![Creating the App Service Plan](docs/media/05-plan.png)

1. Create a Cosmos DB database, and select the Graph (Gremlin) API. 

    ![Create a Cosmos DB with Graph API](docs/media/07-cosmos-graph.png)

1. Create a new Graph in the new Cosmos DB database id named `TweetsDB` and a graph id of `Tweets`. 

    ![Create a new Graph](docs/media/36-create-graph.png)

Keep the browser open! We're going to need some of the keys for use in your app.

### Cloning the demo & linking it with your cloud resources

1. Open the terminal. Run the following commands to clone this repository. 

    ```
    git clone https://github.com/pjmeyer/SmartHotel360-Sentiment-Analysis-App -b scale-lab
    ```

    Now type `cd Smart` followed by the `tab` key (to auto-complete) to enter the repo directory. Type:
    ```
    code .
    ``` 
    
    to open this directory in Visual Studio Code. You may now close the terminal.

1. You'll notice that VS Code will display recommended extensions for your workspace. This is a way for the owner of a repo to indicate which VS Code extensions they recommend you use to get the best development experience. Click on `install all`, and after all the extensions have installed, click `reload` to reload the editor window.

1. Sign in to your Azure subscription by using `F1` (Windows, Linux) or `Cmd-Shift-P` (Mac) to open the Visual Studio Code command palette. Type `Azure` and find the `Azure: Sign In` commannd to sign in to your Azure account. 

    ![Sign in](docs/media/20-signin.png)

1. By clicking **Copy and Open**, your browser will open and allow you to paste in the authentication code. Use the credentials included in your lab. 

    ![Copy and open](docs/media/21-copy-and-open.png)


1. Now that you're logged in to your Azure subscription, you will see the resources you created earlier in the various Azure resource explorers in Visual Studio Code. Next, we'll congifure this code to use the remote resources you created.

    ![Azure explorer windows](docs/media/22-explorers.png) 


1. Copy your Bing Maps API key from the Azure Portal and paste it into the `client/js/webConfig.js` file to the value of the `mapQueryKey` property. 

    ![Bing Maps API Key](docs/media/23-bing-key.png)

1. Copy the Cosmos DB Graph API database's Gremlin Endpoint (found in `Overview`, you'll need to delete the "https://" and ":443/" parts of the string) and Primary Key (found in `Keys`) into the `util/dbconfig.js` file's `endpoint` and `primarykey` properties. 

    ![Configure the Web Site DB](docs/media/42-configure-website.png)

1. Open the integrated terminal by opening the command palette (F1) and searching for `Toggle Integrated Terminal`.

1. Log in to the container registry you created in Azure setup above using the integrated terminal command below. The login server, username, and password will be found in the `access keys` section of the registry you created. 

    Type the integrated terminal, using the credentials for your registry:
    ```
    docker login -u {username} -p {password} {login server}
    ```

1. In the integrated terminal, run `npm install`.

1. In the integrated terminal, run `node util/dbUtil.js` to populate the database with tweets. If you see "all done", you're ready to go!
    > **_Why not use real tweets?_** We could use real tweets! But we'd need to register an API call under your personal Twitter handle to do the search, and we didn't want to make that a requirement. Plus, there are people out there who aren't on Twitter. \*gasp\*


## The SmartHotel web site
### Debugging the site
The web site with this lab enables the employees of SmartHotel360 to see the social sentiment of hotels in the New York City area. A clickable map shows the hotels as green or red circles that, when clicked, show the general Twitter sentiment analysis of each hotel. 

![The clickable map](docs/media/43-map.png)

1. Hit `F5` to launch the web site in the Visual Studio Code debugger. Once the debugger launches, the site will be available at [http://localhost:3000](http://localhost:3000). 

    When the site launches, click some of the circles to see the sample data. You may notice something is wrong! There's a visualization missing in the bottom left. 

1. Go back to VS Code, and open `app.js`. Because VS Code is built on TypeScript, you can use the TypeScript compiler to infer and check types, even on plain JavaScript files!  Let's do that.

    At the top of the file, add `//@ts-check`. 

1. Scroll down, you shoudl see some new squiggles. Hover over the `s` variable. That's odd: 's' should be a tweet object, not a string. Put a breakpoint on `line 35`. Now reload the page in the browser, and you should see VS Code hit on the breakpoint. Hover over 's' again.

1. Aha! 's' is zero, when it should be a tweet object. This is because in the loop above, the keyword `in` is assigning the variable 's' to the index of the array, and not the object that corresponds to that index. Confirm this by typing `data.tweets[s]` into the debug console below. 

1. In the loop, change the word `"in"` to `"of"`, save the file, remove the breakpoint, and click the `reload icon` button near the top of the VS Code editor. Refresh the web page. It's fixed! You should see the new graphic in the bottom left. Back in VS Code, stop the debugger by clicking the `stop icon`. 


### Dockerize the site and push it to Azure Container Registry

The Docker tools for Visual Studio Code make it easy to work with Docker images and containers. This repo has a `Dockerfile` in it.

1. Open the Dockerfile, and right click in the editor window. Select `Build Image`. You'll have the option to change the name, but leave the default and hit `Enter`.

1. Once the image is built you will see the output in the Visual Studio Code terminal. You can also see that the image is now visible in the **Images** node of the Docker Explorer. 
    
    ![Docker image built](docs/media/46-image-built.png)

    > Note: The screen shot above also shows the `node:alpine` image, but the version may vary over time. What's important is that you see the `SmartHotel360-Sentiment-Analysis-App:latest` image in the list. 

1. Now let's tag this image to map it to your Azure Container Registry. Right click on `SmartHotel360-Sentiment-Analysis-App:latest` image, and select `Tag Image`. In the command palette, add the URL of your registry to the beginning of your image name, followed by a "/". It'll look something like this:

    ```
    <yourname>.azurecr.io/SmartHotel360-Sentiment-Analysis-App:latest
    ```

    > Can't find the URL? In the Azure portal, navigate to your registry, and look under the **Access keys** menu option. 

    ![Access keys](docs/media/47-access-keys.png)

1. Right click on your new image, and select `Push`. The terminal window will provide detailed status as the image is pushed to Azure Container Registry. Once it is complete, you can refresh the Docker Explorer to see the image has been published to the ACR resource. 

    ![Pushed Image](docs/media/48-pushed.png)

### Deploy the image to App Service

The final step in the deployment process is to deploy the Docker image from Azure Container Registry to Azure App Service. This can all be done within Visual Studio Code, and the site will be live!

1. Under `Registries -> Azure`, you'll see your registry in the tree. Continue to navigate until you see the `SmartHotel360-Sentiment-Analysis-App:latest` image. Right-click it and select the **Deploy Image to Azure App Service** option. 

    ![Deploy the Image](docs/media/49-deploy-image.png)

1. Select the Resource Group where you created the App Service Plan earlier. 

    ![Select group](docs/media/50-select-group.png)

1. Select the App Service Plan you created earlier. 
    
    ![Select plan](docs/media/51-select-plan.png)

1. Give the new App Service a name. This will need to be a globally unique name that will be used as the prefix to a URL ending with `azurewebsites.net`.

    ![Name the app](docs/media/52-name-app.png)

1. Visual Studio Code will publish the App Service from the Azure Container Registry image in a few seconds. By `ctrl` or `cmd`-clicking the URL in the output window, you can load the app in your browser. 

    ![App published](docs/media/53-app-published.png)

1. Now the app is running, and you can see the social sentiments in the map. 

    ![App running](docs/media/54-app-running.png)

## Conclusion
You did it! You cloned, debugged, and fixed a web app, containerized it, then pushed it to a private container registry, where you then deployed it to Azure App Service PaaS.

For more information on the services you used, check out the web sites below:
- https://code.visualstudio.com
- https://azure.microsoft.com/en-us/services/app-service/containers/
- https://azure.microsoft.com/en-us/services/cosmos-db/