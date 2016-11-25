# Environments

Metadata of regular.express environments.

This includes list of testing environments. It's information of testing group and each test environments in the group. 



Group information is placed `_info.json` in each folder. This file is provided in the following form.

```json
{	
	"title": "C++",
	"description": "C++ is a general-purpose programming language. It has imperative, object-oriented and generic programming features, while also providing facilities for low-level memory manipulation.",
	"website": "https://isocpp.org/",
	"icon": {
		"type": "devicon",
		"value": "cplusplus"
	}
}
```

This information is used in each environment page of [regular.express](http://regular.express) website. Icons are images representing the environment. `type` key supports `devicon` or `img`. Devicon is a set of icons representing programming languages, and provided [website](http://devicon.fr). regular.express uses it, so `value` can be devicon's icon name (e.g. cplusplus, apache, java and so on). If icon type is `img`, value shoud be a image url (e.g. http://some.domain/icon.png)



Each environment's file in the group should be placed in same folder with `_info.json`. Environment file includes parameter to test. This file is provided in the following form. 

```json
{
	"title": "Title of test environment",
	"description": "Description of this test",
	"author": "whoever",
	"docker_image": "repository/project:tag",
	"properties": [
		{
			"name": "parameter_name",
			"type": "string",
			"example": "(0[0-9]{2})-([0-9]{3,4})-([0-9]{3,4})",
			"help" : "help for this parameter",
			"required" : true
		},
      ...
	]
}
```

This includes title, description, author and docker image for testing environment. This metadata is shown in UI directly. 
Parameter is placed in `properties` value as array. Parameter's `name` is shown in UI and also used to send as parameter to image. We support `type` as `string`, `boolean`, `hidden`, `list`. `hidden` type isn't shown in the UI but is sent as specific value that is in json. `list` parameter works like 'select' tag in html. It is shown as combobox in the UI. `example` is example of that parameter as written. It is shown below the  parameter. `help` parameter is document for that parameter. Parameter's help tooltip shows it. If `required` parameter is true, UI shows it as default but if it is false, it is hidden unless click Advanced Mode. 


If type is boolean, default value can be set as following.


```json
{
	"name": "parameter_name",
	"type": "boolean",
	"default": true  
}
```


And if type is list, detail form is following format.


```json
{
	"name": "test_subtype",
	"type": "list",
	"list": {
		"match": "Match",
		"extract": "Extract"
	},
	"default": "match"
}
```


List of value is in `list` as object's key/value pair. Key is internal value that used to send as parameter. But UI just shows the it's value. This also can be set default value. It shoud be one of key list.

If `type` is `hidden`, it is hidden in UI but it is sent as specific `value`. In this, 'match' string will be sent as 'test_type' parameter.


```json
{
	"name": "test_type",
	"type": "hidden",
	"value" : "match"
}
```



All of these parameters and input dataset are encoded to json before test. For example, parameter can be encoded as following.

```json
{
	"test_type": "match",
  	"regex": "([0-9]+)",
  	"icase": true
}
```


The input dataset is encoded in a json array as following.


```json
["123", "test", "012, "b"]
```


Then the parameter and dataset json are each encoded as base64 to send to docker container. We use base64 to avoid preprocessing special character in different operating system. So docker command for this test is following.


```
$ docker run reposotory/project:tag ew0KCSJ0ZXN0X3R5cGUiOiAibWF0Y2giLA0KICAJInJlZ2V4IjogIihbMC05XSspIiwNCiAgCSJpY2FzZSI6IHRydWUNCn0=  WyIxMjMiLCAidGVzdCIsICIwMTIsICJiIl0=
```


So docker container should decode parameter encoded as base64, and parse json string. Then it will use the parameter and dataset for executing regex testcode. Docker image we provide uses `base64` command to decode base64 parameter in the bash shell. See this bash shell code:


```shell
arg=();
for var in "$@";
 do arg+=("$(echo -n "$var" | base64 -d)"); 
 done; 
path_of_testprogram "${arg[@]}"
```


This bash shell decodes the base64 parameters passed in the docker command and passes them to the test program.



After executing program, It should return json for test result as following format. 


```json
{
	"type": "type of test result.",
 	"result": {
		"resultList": [
			...
		]
 	},
	"debugOutput": "If this exists, UI show this for debugging.",
	"exception": "This only uses when test program raises exception. If this exists, other data is ignored."
}
```



`type` can be three value as `STRING`, `MATCH`, `GROUP`. Each of this format is following.

`STRING` type:

```json
{
	"type": "STRING",
 	"result": {
		"resultList": [
			"String Result1",
			"String Result2",
			...
		]
 	}
}
```

In this case, `resultList` is just string array for each result of test record.



`MATCH` type:

```json
{
	"type": "MATCH",
 	"result": {
		"resultList": [
			true,
			false,
			...
		]
 	}
}
```

This type indicates whether each record matches a regular expression. This is same to `STRING` result except result of each record is boolean type.



`GROUP` type:

```json
{
	"type": "GROUP",
	"columns": ["Group 0", "Group 1"]
 	"result": {
		"resultList": [
			{
				"list": [
					["Of the first matching in the first record, Captured Group 0", "Group 1"],
					["Of the second matching in the first record, Captured Group 0", "Group 1"]
				]
			},
			{
				"list": [
					["Of the first matching in the second record, Captured Group 0", "Group 1"],
					["Of the second matching in the second record, Captured Group 0", "Group 1"]
				]
			},
			...
		]
 	}
}
```

Group type is complex a little bit. This type is for representing the captured group information in sequential matching. Serveral language supports this. `resultList` is matching results for each record. One record can include serveral matching information. For this, each result of records is object that has `list` value. The `list` includes all of matching for that record. For example, first record matches twice sequencially to regex that has two group, `list` value has two match results with two captured group information. If alias of captured group exists, this can be set in `columns` value as name array.



For reading these result json, we use standard output stream of test program. To distinguish result json and other output in program, we uses special phrase for this. We just capture json result between ##START_RESULT## and ##END_RESULT##. So we ignore other output not in the phrases. Test docker image should print json result that is placed between that phrases. For example:

```
...
##START_RESULT##
{
	"type": "MATCH",
 	"result": {
		"resultList": [
			true,
			false
		]
 	}
}
##END_RESULT##
..
```



All of docker image that we provide follow these rule. If you want to add new environment, deploy docker image follows these rule to docker hub, and write environment json and `_info.json` in specific folder. Then request us. Please contribute to us for add new environment. We'll check it and push environment file to this repository.
