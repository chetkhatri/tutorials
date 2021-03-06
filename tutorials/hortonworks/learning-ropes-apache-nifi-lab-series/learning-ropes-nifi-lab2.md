---
layout: tutorial
title: Lab 2 Enhance the DataFlow with Geo Location Enrichment
tutorial-id: 640
tutorial-series: Basic Development
tutorial-version: hdp-2.4.0
intro-page: false
components: [ nifi ]
---

# Lab 2: Enhance the DataFlow with Geo Location Enrichment

## Introduction

In this section, you will build a geographic location enrichment for the vehicle filtering dataflow. You will obtain a deep understanding of the automated and managed flow of information between multiple systems and the facilities in NiFi for monitoring and examining the dataflow. To add this enhancement, we will incorporate Google Places Nearby API with NiFi, which will show the neighborhoods nearby the vehicle's location as it moves. This incorporation of external API's is a feasible design pattern because NiFi makes it easy to bring in other technologies to process the data.

In this lab, you will build the Geo Location Enrichment section of the dataflow:

![complete_dataflow_lab2_geoEnrich](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/complete_dataflow_lab2_geoEnrich.png)

Feel free to download the [Lab2-NiFi-Learn-Ropes.xml](https://raw.githubusercontent.com/hortonworks/tutorials/hdp/assets/learning-ropes-nifi-lab-series/lab2-template/Lab2-NiFi-Learn-Ropes.xml) template file or if you prefer to build the dataflow from scratch, continue on to the lab.


## Pre-Requisites
- Completed Lab 0: Download, Install and Start NiFi
- Completed Lab 1: Build A Simple NiFi DataFlow

## Outline
- Google Places API
- Step 1: Obtain API Key for NiFi to Build HTTP URL
- Step 2: Build Geo Location Enrichment DataFlow Section
- Step 3: Run NiFi DataFlow
- Summary
- Further Reading

## Google Places API

Google Places API Web Service returns information about places: establishments, geographic locations and prominent points of interest using HTTP requests. The Places API includes six place requests: Place Searches, Place Details, Place Add, Place Photos, Place Autocomplete and Query Autocomplete. Read more about these place requests in [Introducing the API](https://developers.google.com/places/web-service/intro).

All requests are accessed through an HTTP request and return either JSON or XML response.

What are the necessary components to use the Places API?
- https:// protocol
- API Key

### Step 1: Obtain API Key for NiFi to Build HTTP URL

For our dataflow, our task is to enrich the data by searching for neighborhoods within proximity of a vehicle's varying location. We will retrieve two parameters from this data: name of the neighborhoods and San Francisco Muni Transit Agency. So, we will integrate Nearby Search HTTP request with NiFi.

The Nearby Search request is an HTTP URL of the following definition, which we will need for NiFi:

~~~
https://maps.googleapis.com/maps/api/place/nearbysearch/output?parameters
~~~

The `output` can come in two formats: `json` or `xml`. We will use json for this lab.

Let's obtain the **required parameters** to initiate a Nearby Search request.

1\. We will need to [obtain an API key](https://developers.google.com/places/web-service/get-api-key), so it can identify our application for quota management and places added from the application are instantly available to our app (NiFi).

2\. We will use a standard Google Places API. Click on the blue **Get A Key** button to activate the API Web Service.

3\. A window will appear that says **Select a project where your application will be registered**. In the dropdown menu, you should have `Create a new project` as the option. We will create a project for this application. Click on **Continue**. Wait a few seconds for the new window to load.

4\. A screen with **Server key 1** in the Name field will appear, click on the blue **Create** button.

5\. Now a screen with your unique API key in a table will appear similar to the Table below:

**Table 1: Example of the API Key Table**

| Name  | Creation date  | Type  | Key  |
|:---|:---:|:---:|---:|
| `Server Key 1`  | `June 14, 2016`  | `Server`  | `AIzaSyDY3asGAq-ArtPl6J2v7kcO_YSRYrjTFug`  |

Now we have the API Key parameter for our HTTP request. We also have the other required parameters: **location** thanks to lab 1 in which we extracted longitude & latitude attributes and **radius**, which can be a distance that does not surpass 50,000 meters. We will use one optional parameter **type** to signify what type of place we are interested in searching for.

6\. Let's build our HTTP URL with the parameters below, so we can insert the URL as a property value into **InvokeHTTP** later in the lab.

- API Key = AIzaSyDY3asGAq-ArtPl6J2v7kcO_YSRYrjTFug
- Latitude = ${Latitude}
- Longitude = ${Longitude}
- radius = 500
- type = neighborhood

~~~
https://maps.googleapis.com/maps/api/place/nearbysearch/json?location=${Latitude},${Longitude}&radius=500&type=neighborhood&key=AIzaSyDY3asGAq-ArtPl6J2v7kcO_YSRYrjTFug
~~~

> Note: Your API Key will be different than the one in the URL above.


### Step 2: Build Geo Location Enrichment DataFlow Section

Six processors are needed to add geographic location enrichment to your dataflow. Each processor holds a critical role in transporting the enriched data to a destination:

- **InvokeHTTP** performs an HTTP request to access data from Google Places API about places nearby a vehicle's location
- **EvaluateJsonPath** extract neighborhoods_nearby and city data elements out of JSON structure
- **RouteOnAttribute** routes FlowFiles as long as their neighborhoods_nearby and city attributes do not contain empty strings
- **AttributesToJSON** represents the enriched attributes in JSON structure
- **MergeContent** merges FlowFiles together by concatenating their JSON content together
- **PutFile** writes the enriched geographic location contents of the FlowFile to the local file system

### 2.1 Learning Objectives: DataFlow Geo Enrichment Addition
- Add/Configure/Connect processors to ingest, filter and store geo enriched API data
- Troubleshoot problems that may occur
- Run the dataflow

### InvokeHTTP

1\. Add the InvokeHTTP processor onto the NiFi graph. Connect **RouteOnAttribute** from lab1 to **InvokeHTTP** processor. When the Create Connection window appears, verify **Filter Attributes** checkbox is checked, if not check it. Click **Add**.

2\. Open InvokeHTTP configure properties tab and add the property listed in **Table 1**.

**Table 1:** Update InvokeHTTP Property Value(s)

| Property  | Value  |
|:---|---:|
| `Remote URL`  | `https://maps.googleapis.com/maps/api/place/nearbysearch/json?location=${Latitude},${Longitude}&radius=500&type=neighborhood&key=AIzaSyDY3asGAq-ArtPl6J2v7kcO_YSRYrjTFug` |

![invokeHTTP_config_property_tab_window](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/invokeHTTP_config_property_tab_window.png)

3\. Navigate to the **Settings** tab, change the name from InvokeHTTP to `GoogleNearbySearchAPI`. Under Auto terminate relationships check the **Failure**, **No Retry**, **Original** and **Retry** checkboxes. Click **Apply** button.

### EvaluateJsonPath

1\. Add the EvaluateJsonPath processor onto the NiFi graph. Connect InvokeHTTP to EvaluateJsonPath processor. When the Create Connection window appears, select **Response** checkbox. Click Apply.

2\ Open EvaluateJsonPath configure properties tab and update the original properties with the properties listed in **Table 2**. Note: add `city` and `neighborhoods_nearby` property by clicking the **New property** button, then insert their values into the properties tab.

**Table 2:** Update and Add New EvaluateJsonPath Property Values

| Property  | Value  |
|:---|---:|
| `Destination`  | `flowfile-attribute`  |
| `Return Type`  | `json`  |
| `city`  | `$.results[0].vicinity`  |
| `neighborhoods_nearby`  | `$.results[*].name`  |

![evaluateJsonPath_config_property_tab_window](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/evaluateJsonPath_config_property_tab_window.png)

3\. Navigate to the **Settings** tab. Under Auto terminate relationships check the **unmatched** and **failure** checkboxes. Click **Apply** button.


### RouteOnAttribute

1\. Add the EvaluateJsonPath processor onto the NiFi graph. Connect EvaluateJsonPath to RouteOnAttribute processor. When the Create Connection window appears, select **matched** checkbox. Click Apply.

2\. Open RouteOnAttribute configure properties tab and click on **New property** button to add `RouteNearbyNeighborhoods` to property name and insert its NiFi expression value listed in **Table 3**.

**Table 3:** Add New RouteOnAttribute Property Value

| Property  | Value  |
|:---|---:|
| `RouteNearbyNeighborhoods`  | `${city:isEmpty():not():and(${neighborhoods_nearby:isEmpty():not()})}`  |

![routeOnAttribute_geoEnrich_config_property_tab_window](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/routeOnAttribute_geoEnrich_config_property_tab_window.png)

3\. Navigate to the **Settings** tab, change the name from RouteOnAttribute to `RouteNearbyNeighborhoods`. Under Auto terminate relationships check the **unmatched** checkbox. Click **Apply** button.

### AttributesToJSON

1\. Add the EvaluateJsonPath processor onto the NiFi graph. Connect RouteOnAttribute to AttributesToJSON processor. When the Create Connection window appears, select **RouteNearbyNeighborhoods** checkbox. Click Apply.

2\. Open AttributesToJSON configure properties tab and update the properties with the information listed in **Table 4**.

**Table 4:** Update AttributesToJSON Property Values

| Property  | Value  |
|:---|---:|
| `Attributes List`  | `Vehicle_ID, city, Latitude, Longitude, neighborhoods_nearby, Last_Time`  |
| `Destination`  | flowfile-content  |

![attributesToJSON_geoEnrich_config_property_tab_window](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/attributesToJSON_geoEnrich_config_property_tab_window.png)

3\. Navigate  to the **Settings** tab, under Auto terminate relationships check the **failure** checkbox. Click **Apply** button.

### MergeContent

1\. Add the MergeContent processor onto the NiFi graph. Connect AttributesToJSON to MergeContent processor. When the Create Connection window appears, select **success** checkbox. Click Apply.

2\. Open MergeContent configure properties tab and update the properties with the information listed in **Table 5**. For the Demarcator property, type `,` then press `shift+enter`.

**Table 5:** Update MergeContent Property Values

| Property  | Value  |
|:---|---:|
| `Minimum Number of Entries`  | `20`  |
| `Maximum Number of Entries`  | `25`  |
| `Delimiter Strategy`  | Text  |
| `Header`  | `[`  |
| `Footer`  | `]`  |
| `Demarcator`  | `,` {press-shift+enter}  |

![mergeContent_geoEnrich_config_property_tab_window](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/mergeContent_geoEnrich_config_property_tab_window.png)

3\. Navigate  to the **Settings** tab, under Auto terminate relationships check the **failure** and **original** checkbox. Click **Apply** button.

### PutFile

1\. Add the PutFile processor onto the NiFi graph. Connect MergeContent to PutFile processor. When the Create Connection window appears, select **merged** checkbox. Click Apply.

2\. Open PutFile configure properties tab and update the property with the information listed in **Table 6**.

**Table 6:** Update PutFile Property Values

| Property  | Value  |
|:---|---:|
| `Directory`  | `/tmp/nifi/output/nearby_neighborhoods_search`  |

![putFile_geoEnrich_config_property_tab_window](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/putFile_geoEnrich_config_property_tab_window.png)

3\. Navigate  to the **Settings** tab, under Auto terminate relationships check the **success** checkbox. Click **Apply** button. Connect the processor to itself and when the Create Connection window appears, select **failure** checkbox.


### Step 3: Run NiFi DataFlow

Now that we added geographic location enrichment dataflow section to our previous dataflow, let's run the dataflow and verify if we receive the expected results in our output directory.

1\. Go to the actions toolbar and click the start button ![start_button_nifi_iot](/assets/learning-ropes-nifi-lab-series/lab1-build-nifi-dataflow/start_button_nifi_iot.png). Your screen should look like the following:

![complete_dataflow_lab2_geoEnrich](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/complete_dataflow_lab2_geoEnrich.png)

2\. Let's check the data was written to our expected directory, open your terminal. Make sure to SSH into your sandbox if on sandbox, else navigate to the output directory on your local machine. Navigate to the directory you wrote for the PutFile processor. List the files and open one of the newly created files to view geographic neighborhoods nearby transit location enrichment data output. In the tutorial our directory path is: `/tmp/nifi/output/nearby_neighborhoods_search`.

~~~
cd /tmp/nifi/output/nearby_neighborhoods_search
ls
vi 38997303004413
~~~

![output_geoEnrich_nearby_neighborhoods_data](/assets/learning-ropes-nifi-lab-series/lab2-geo-location-enrichment-nifi-lab-series/output_geoEnrich_nearby_neighborhoods_data.png)


## Summary

Congratulations! For the Geo Enrichment section of the dataflow, you learned to use InvokeHTTP to access geographic location of nearby places with Google Places Search API. You learned to add NiFi expression variables into InvokeHTTP property RemoteURL, so that the values for latitude and longitude constantly change in the URL when new FlowFiles pass through this processor. You learned to use EvaluateJsonPath similar to EvaluateXPath, except JSON Expression is used to extract JSON elements (neighborhoods_nearby & city) from a JSON structure. Now you know how to incorporate external API's into NiFi further enhance the dataflow.

## Further Reading

- [Google Places API](https://developers.google.com/places/)
- [HTTP Protocol Overview](http://code.tutsplus.com/tutorials/http-the-protocol-every-web-developer-must-know-part-1--net-31177)
