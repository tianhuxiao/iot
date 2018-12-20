# QuerySuperDeviceGroup {#reference_xxb_ss4_1gb .reference}

Call this operation to query the parent group of a subgroup.

## Request parameters {#section_itm_ws4_1gb .section}

|Parameter|Type|Required|Description|
|:--------|:---|:-------|:----------|
|Action|String|Yes|The operation to be performed. Set the value to QuerySuperDeviceGroup.|
|GroupId|String|Yes|The unique identifier of the subgroup.|
|Common Request Parameters|-|Yes|See [Common parameters](reseller.en-US/Developer Guide (Cloud)/API reference/Common parameters.md#).|

## Response parameters {#section_t2p_ht4_1gb .section}

|Parameter|Type|Description|
|:--------|:---|:----------|
|RequestId|String|The globally unique ID generated by Alibaba Cloud for the request.|
|Success|Boolean|Indicates whether the call is successful. A value of true indicates that the call is successful. A value of false indicates that the call has failed.|
|ErrorMessage|String|The error message returned when the call fails.|
|Code|String|See [Error Codes](reseller.en-US/Developer Guide (Cloud)/API reference/Error codes.md#).|
|Data|Data|The parent group information returned when the call is successful. For more information, see the following table GroupInfo.|

|Parameter|Type|Description|
|:--------|:---|:----------|
|GroupName|String|The parent group name.|
|GroupId|String|The unique identifier of the parent group.|
|GroupDesc|String|The description of the parent group.|

## Examples {#section_v2d_154_1gb .section}

**Request example**

```
https://iot.cn-shanghai.aliyuncs.com/?&Action=QuerySuperDeviceGroup
&GroupId=DMoI2Kby5m62Sirz
&Public Request Parameters
```

**Response example**

JSON format

```
{
    "Data":{
        "GroupName":"IOTTEST",
        "GroupId":"tDQvBJqbUyHskDse",
	"GroupDesc":"A test."
    },
    "RequestId":"7411716B-A488-4EEB-9AA0-6DB05AD2491F",
    "Success":true
}
```

XML format

```
<? xml version="1.0" encoding="UTF-8" ? >
<QuerySuperDeviceGroupResponse>
	<Data>
	    <GroupName>IOTTEST</GroupName>
	    <GroupId>tDQvBJqbUyHskDse</GroupId>
	    <GroupDesc>A test. </GroupDesc>
	</Data>
	<RequestId>7411716B-A488-4EEB-9AA0-6DB05AD2491F</RequestId>
	<Success>true</Success>
</QuerySuperDeviceGroupResponse>
```
