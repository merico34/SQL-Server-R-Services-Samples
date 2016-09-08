<img src="../Images/management.png" align="right">
<h1>Campaign Management:
Typical Workflow Walkthrough</h1>

To demonstrate a typical workflow, we'll introduce you to a few personas.  You can follow along by performing the same steps for each persona.  

NOTE: If you’re just interested in the outcomes of this process we have also created a fully automated solution that simulates the data, trains and scores the models by executing PowerShell scripts. This is the fastest way to deploy. See [PowerShell Instructions](Powershell_Instructions.md) for this deployment.

## Server Setup and Configuration with Danny the DB Admin

Let me introduce you to  Danny, the DB Admin. It is Danny's job to configure and maintain the SQL Server that stores all the historical data about campaigns at our insurance company.  

Danny was responsible for installing and configuring the SQL Server, as well as configuring the ODBC connection between SQL and PowerBI.  You can perform these steps by using the instructions in <a href="START_HERE.md">START HERE</a>. 

## Data Prep and Modeling with Debra the Data Scientist

Now let's meet Debra, the Data Scientist. Debra's job is to use historical data to predict a model for future campaigns. She will actually create two models and compare them, then use the one she likes best to compute a prediction for each combination of day, time, and channel for each lead, and then select the combination with the highest probability of conversion - this will be the recommendation for that lead.  

Debra will work on her own machine, using  using  [R Client](https://msdn.microsoft.com/en-us/microsoft-r/install-r-client-windows) to execute these R scripts. She will need to [install and configure an R IDE](https://msdn.microsoft.com/en-us/microsoft-r/r-client-get-started#configure-ide) to use with R Client.  

Now that Debra's environment is set up, she  opens her IDE and performs the following:

1.  First she installs the packages that she'll need.  She executes the following code in R:

    ```
    install.packages("data.table")
    install.packages("ROCR")
    ```

2.  Now she'll develop R scripts to prepare the data.  To view the scripts she writes, open the following files from the CampaignManagement/R directory:

    a.	**step1_input_data.R**:  We'll sneak this script in first - in a real scenario the data would already be present on the SQL Server.  However, we're simulating data in this solution packet, and this script does the simulation. 

    b.	**step2_data_preprocessing.R**: Performs preprocessing steps -- outlier treatment and missing value treatment on the input datasets.

    c.	**step3_feature_engineering_AD_creation.R**:  Performs Feature Engineering and creates the Analytical Dataset.   Feature Engineering consists of creating new variables in the market touchdown dataset by aggregating the data in multiple levels.  The table is aggregated at a lead level, so variables like channel which will have more than one value for each user are pivoted and aggregated to variables like SMS count, Email count, Call Count, Last Communication Channel, Second Last Communication Channel etc.

3.  If you are following along, you will need to replace the connection string at the top of each file with details of your login and database name in each file.  For example:
   
    ```
    connection_string <- "Driver=SQL Server;Server=myServerName;Database=CampaignManagement;UID=rdemo;PWD=D@tascience"
    ```
    
    This connection string contains all the information necessary to connect to the SQL Server from inside the R session. As you can see in the script, this information is then used in the `RxInSqlServer()` command to setup a `sql` string.  The `sql` string is in turn used in the `rxSetComputeContext()` to execute code directly on the SQL Server machine.  You can see this in the .R files:

    ```
    connection_string <- "Driver=SQL Server;Server=[SQL Server Name];Database=[Database Name];UID=[User ID];PWD=[User Password]"
    sql <- RxInSqlServer(connectionString = connection_string)
    ...
    rxSetComputeContext(sql)
        ```
    
 4.  After running the first three scripts, Debra goes to SQL Server Management Studio to log in and view the results of feature engineering by running the following query in SSMS.
  ```
  SELECT TOP 1000 [Lead_Id]
    ,[Sms_Count]
    ,[Email_Count]
    ,[Call_Count]
    ,[Last_Channel]
    ,[Second_Last_Channel]
  FROM [CampaignManagement].[dbo].[market_touchdown_agg]
  ```
5.  Now she is ready for training the models.  She creates and executes the script you can find in **step4_model_rf_gbm.R**. Again, remember to replace the `connection_string` value with your information at the top of the file before you run this yourself.)  

6.  The above script (**step4_model_rf_gbm.R**) also scores data for leads to be used in a new campaign. The code uses the champion model to score each lead multiple times - for each combination of day of week, time of day, and channel - and selects the combination with the highest probability to convert for each lead.  This becomes the recommendation for that lead.  The scored datatable shows the best way to contact each lead for the next campaign. The recommendations in this table (`Lead_Scored_Dataset`) are used for the next campaign the company wants to deploy.

7.  Debra will now use PowerBI to visualize the recommendations created from her model.  She creates the PowerBI Dashboard which you can find in the **Resources** directory.  She uses an ODBC connection to connect to the data, so that it will always show the most recently modeled and scored data, using the [instructions here](Visualize_Results.md).

<img src="../Images/visualize.png">

The dashboard file is included in the **Resources** directory.


## Operationalize with Debra and Danny

Debra has completed her tasks.  She has connected to the SQL database, executed code both locally and on the SQL machine to clean the data, create new features, train two models and select the champaion model. She's scored data, created recommendations, and also created a summary report which she will hand off to Bernie - see below.

While this task is complete for the current set of leads, our company will want to perform these actions for each new campaign that they deploy.  Instead of going back to Debra each time, Danny can operationalize the code in TSQL files which he can then run himself each month for the newest campaign rollouts.

Debra hands over her scripts to Danny who adds the code to the files you can see in the **CampaignManagement\\SQL** directory. He'll test out the start to finish set of TSQL code by running them - you can do this yourself by following the directions in the [SQLR Instructions](SQLR_Instructions.md).

Finally, Danny may want to automate the process even more by developing the PowerShell scripts to invoke the TSQL files.  You can find these scripts in the **CampaignManagement** directory, and execute them yourself by following the See [PowerShell Instructions](Powershell_Instructions.md).  As noted earlier, this is the fastest way to execute all the code included in this solution.

A summary of this process and all the files involved is described in more detail [here](../data-scientist.md).


## Deploy and Visualize with Bernie the Business Analyst 

Now that the predictions are created and the recommendations have been saved, we will meet our last persona - Bernie, the Business Analyst. Bernie will use the Power BI Dashboard to learn more about the recommendations (first tab).

Bernie will then let the Campaign Team know that they are ready for their next campaign rollout - the data in the `Lead_Scored_Dataset` table contains the recommended time and channel for each lead in the campaign.  The team uses these recommendations to contact leads in the new campaign.