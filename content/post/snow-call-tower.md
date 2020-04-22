---
title: "Call Ansible Tower from ServiceNow"
date: 2019-08-05T11:49:05+01:00
draft: false
tags: [ansible,tower,servicenow]
---

In this post we will look at how we can call Ansible Tower from ServiceNow as part of a ServiceNow Catalog Request. For this example, I have a [playbook](https://github.com/pharriso/ansible_network_demo/blob/master/bigip_pool_member_snow.yml) in Ansible Tower that will manage the membership of a node within an F5 loadbalancer pool. The playbook expects the user to input the name of the node that should be managed and the state it should be in - either enabled or forced_offline. So we need to pass these variables from ServiceNow to Ansible Tower. Note that the playbook will also close down the ServiceNow request to fully automate the process without any manual intervention.

This integration allows us to use Tower for what it does best - provide the enterprise platform for executing our Ansible playbooks. We can leave Tower to manage credentials, provide RBAC and capture the audit history. At the same time we can leverage existing processes within ServiceNow like approval processes and general change management workflows.

**NOTE** I am not a ServiceNow developer and this is definitely not a tutorial in how best to use ServiceNow. This is something that I had to work out for a demo around Ansible Tower.

#### Ansible Tower Job Template

I already have my [job template](https://docs.ansible.com/ansible-tower/latest/html/userguide/job_templates.html) defined in Ansible Tower. Make a note of the job ID which is displayed in the URL when you click on the Job Template in Tower. For example the job ID is 10 in this sample URL - **https://tower.example.com/#/templates/job_template/10**

To allow extra variables to be passed into the Ansible job execution, I need to enable **Prompt on launch** for **extra variables** as per the screenshot below. 

![](/images/snow_tower_job_template.png)

#### Tower with self-signed certs

As I am using a self-signed certificate in my instance, I need to configure ServiceNow to not verify the hostname in the certificate. To do this, search for **sys_properties.list** in the filter navigator. Then search for **com.glide.communications.httpclient.verify_hostname** and set it to false.

#### ServiceNow Outbound REST Message

The first thing we need to configure in ServiceNow, is an Outbound REST message. This defines how it will connect to the Ansible Tower API to launch a job. This includes the API call, credentials and any extra variables we might want to pass from ServiceNow to Ansible Tower.  

In ServiceNow, navigate to **System Web Services -> Outbound -> REST Message** and click **New**.

Enter a name and the URL for the REST endpoint. This should be your Ansible Tower URL with the job ID that you want to launch.

![](/images/snow_rest_name.png)

Set the **Authentication type** to **basic** and then click on the magnifying glass next to **Basic Auth profile**. Click **New** and enter a name for credential & the username and password to authenticate to Tower.

![](/images/snow_cred.png)

Next set the Content-Type by selecting the **HTTP Request** tab and then adding a new **HTTP Header**.

![](/images/snow_endpoint_http_request.png)

Click submit and then go back into your REST Message. As mentioned earlier, I want to pass variables from ServiceNow to Ansible Tower so I can use them in my Ansible automation. To do this, I need to add some content to the HTTP Post. Under **HTTP Methods** click **New** and add the details for our POST Request.  The **HTTP Method** needs to be **POST** and the **Endpoint** needs to be the url for the job we are launching.

![](/images/snow_post.png)

Now under the **HTTP Request** tab, add a new **HTTP Query Parameter** with a **Name** of **Content** and a **Value** which contains the variable data I will pass to Tower. In my example:

```bash
{"extra_vars": { "member_name": "${f5_member}", "snow_request": "${snow_request}", "member_state": "${f5_member_state}" } }
```

Note the **${snow_request}** variable here. We will come back to that later.

The **HTTP Query Parameter** should look as follows.

![](/images/snow_extra_vars.png)

Now we have defined how ServiceNow will make an API call to Ansible Tower. We need to create a workflow to define how this API call will be triggered.

#### ServiceNow Workflow

In ServiceNow, navigate to **Workflow -> Editor**. Select **New Workflow**. Name the workflow and set the table to **Requested Item [sc_req_item]**. Press submit and you will be taken into the workflow design canvas.

![](/images/snow_workflow_name.png)

Delete the connector between Begin and End. Then Select the **Core** tab in the right hand pane. Under **Utilities**, drag the **Run Script** object onto the workflow canvas. Name the script and then paste the script contents in.

```bash
try { 
 var r = new sn_ws.RESTMessageV2('Tower Job Launch', 'F5 Member Job');
 r.setStringParameterNoEscape('f5_member', current.variables.f5_member);
 r.setStringParameterNoEscape('f5_member_state', current.variables.f5_member_state);
 r.setStringParameterNoEscape('snow_request', current.request.number);

 var response = r.execute();
 var responseBody = response.getBody();
 var httpStatus = response.getStatusCode();
}
catch(ex) {
 var message = ex.message;
}
```

Things to note:

1. The name of the Outbound REST Message and HTTP Method name - **('Tower Job Launch', 'F5 Member Job')**
2. Example variable that will be passed from Catalog request - **('f5_member', current.variables.f5_member)**
3. This is a special var we are determining from the ServiceNow catalog request. We will pass the request number that is created by the user as a variable called **snow_request** - **('snow_request', current.request.number)**

Now let's drag an approval task onto our workflow. Again, in the **Core** tab, drag the **Approval User** or **Approval Group** item across. Name the approval step and select the user or group that needs to approve. 

Finally, let's connect our workflow up. Drag from small orange box inside **Begin** to your **Approval** task. Then drag from **Approved** to our **Run Script**. Then drag from **Rejected** to **End**. Lastly, drag from the **Script** to **End**. The flow should look as follows. 

![](/images/snow_workflow_image.png)

#### ServiceNow Service Catalog

Now that we have defined out outbound REST call and our workflow, we can add a Service Catalog item to expose this to end users. In ServiceNow, navigate to **Service Catalog -> Catalog Definitions -> Maintain Items**. Click on **New** and enter a name for the Service Catalog item. Select the **Catalogs** you want the item to appear in. I have just selected **Service Catalog** for this example. Then in the **Process Engine** tab, search for your workflow.

![](/images/snow_service_request_workflow.png)

Now we can define the information we want to prompt the user for which will be passed to Ansible Tower as variables. Within your Catalog Item, navigate to the **Variables** tab at the bottom of the screen. Click **New** to define a new variable.

Select the question **Type** e.g. single line text or multi-choice. Then enter the question you want the user to see in the Catalog item in the **Question** field. In the **Name** field, enter the name of the variable that you want to pass to Tower.

![](/images/snow_sc_var.png)

For multi-choice variables, you need to add **Question Choices** at the bottom of this screen. The **Text** being what you want the user to see on the form and the **Value** being the value of the variable that will be passed to Tower.

![](/images/snow_sc_question_choices.png)

#### Executing the workflow

So what should happen when we order this Catalog item? 

1. The user is prompted for the name of the host they want to manage in the F5 loadbalancer and the desired state of that host - enabled or forced_offline.
2. The request is raised and will await approval.
3. Once the request is approved ServiceNow will launch the playbook via the Tower API. The answers that the user provided in the catalog request will be passed as variables to the Ansible Tower playbook. We will also pass the ServiceNow request number.
4. Ansible will manage the host in the F5 loadbalancer pool as requested by the user.
5. Ansible will update the ticket in ServiceNow to say what has happened and close the ticket.

You can watch a recording of this below.

[![snow-tower](/images/snow-video.png)](http://www.youtube.com/watch?v=BcoffUF9Yhg)

#### Acknowledgements

There were two excellent guides I used for this. I just wanted to piece the best bits from both guides together:

https://liveaverage.com/blog/ansible-tower-and-servicenow-integration-in-10-minutes/

https://github.com/eanylin/ansible-lab/tree/master/servicenow_demo/
