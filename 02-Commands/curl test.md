Hi Shubham,

 

Please find  the below curl, could it be any permission issue ?

 



curl --location 'https://ping-uat.apps.pnguat.arabbanking.local/openidm/managed/user?_queryFilter=userName+eq+%22muhammad.saajid%40bank-abc.com%22'

{

    "code": 503,

    "reason": "Service Unavailable",

    "message": "Access Denied"

}

 

 