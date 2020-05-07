# Picobrew Z API

The Z uses JSON in requests and responses.  Responses appear to (almost) always echo back the full request plus some additional fields.  In the documentation below, only the additional fields are specified for the responses, it is assumed that all requests fields are also present unless otherwise noted.

The token id used in URLs is simply the mac address of the Z without any delimiters and with all letters being lowed-cased.

The Authentication header type of Bearer is provided by the Z. It appears the Z persists this token after registration.

## HTTP Header Values on Request

* Authorization: Bearer <unique>
* Content-Type: application/json (for PUT/POST)
* Host: www.picobrew.com
* Cache-Control: no-cache
* Connection: close

No user agent or encoding headers appear to be sent.

## Boot Up Synchronization

This message is sent when the Z initially boots and is used by the device to determine if there are updates which need to be downloaded or brewing sessions which need to be resumed.  Note that if the user receives an indication that a session was in progress but chooses not to resume it, no further message is sent by the Z to the backend.

##### Request

`PUT https://www.picobrew.com/Vendors/input.cshtml?type=ZState&token=<token> HTTP/1.1`

Query String:

* token:  The unique identifier for the Z

JSON Request Body:

* BoilerType:  Known values are:
    * 0 - Big Boiler
    * 1 - Small Boiler
* CurrentFirmware: The current version of the firmware, delimited by periods.

Example:

```
{
    "BoilerType": 1,
    "CurrentFirmware": "0.0.116"
}
```

##### Response

* Alias:  Default is ZSeries, can be renamed by user.
* IsRegistered:  Appears to always be true.
* IsUpdated:  If false, indicates that the UpdateAddress will contain a firmware update address.
* ProgramUri:  Appears to always be null.
* RegistrationToken:  Appears to always be -1.
* SessionStats
   * DirtySessionsSinceClean: The number of sessions since the last cleaning cycle.
   * LastSessionType:  The type of the last session.  (See begin session for known values.)
   * ResumableSessionID:  The ID of the session that was in progress when the machine lost power.
* TokenExpired:  Appears to always be false.
* UpdateAddress:  If no update is available, the value is the integer -1.  If an update is available, this is a string with a URL for a firmware binary to download and install.
* UpdateToFirmware: This value always appears to be null.
* ZBackendError:  Appears to always be 0.

Example 1:  (A typical response)

```
{
    "Alias": "ZSeries",
    "BoilerType": 1,
    "CurrentFirmware": "0.0.116",
    "IsRegistered": true,
    "IsUpdated": true,
    "ProgramUri": null,
    "RegistrationToken": "-1",
    "SessionStats": {
        "DirtySessionsSinceClean": 1,
        "LastSessionType": 5,
        "ResumableSessionID": -1
    },
    "TokenExpired": false,
    "UpdateAddress": "-1",
    "UpdateToFirmware": null,
    "ZBackendError": 0
}
```

Example 2:  (A firmware update is required)

```
{
    "Alias": "ZSeries",
    "BoilerType": 1,
    "CurrentFirmware": "0.0.87",
    "IsRegistered": true,
    "IsUpdated": false,
    "ProgramUri": null,
    "RegistrationToken": "-1",
    "SessionStats": {
        "DirtySessionsSinceClean": 0,
        "LastSessionType": 1,
        "ResumableSessionID": -1
    },
    "TokenExpired": false,
    "UpdateAddress": "http://picobrewcontent.blob.core.windows.net/firmware/zseries/zseries_0_0_116.bin",
    "UpdateToFirmware": null,
    "ZBackendError": 0
}
```

Example 3:  (A session is available to be resumed)

```
{
    "Alias": "ZSeries",
    "BoilerType": 1,
    "CurrentFirmware": "0.0.116",
    "IsRegistered": true,
    "IsUpdated": true,
    "ProgramUri": null,
    "RegistrationToken": "-1",
    "SessionStats": {
        "DirtySessionsSinceClean": 2,
        "LastSessionType": 6,
        "ResumableSessionID": 60xxx
    },
    "TokenExpired": false,
    "UpdateAddress": "-1",
    "UpdateToFirmware": null,
    "ZBackendError": 0
}
```

## Fetch Recipe Summary

Fetches a list of all of the recipes from the server.  It appears that "pagination" of the recipes is supported through setting a value for offset. This appears to only be used when the Kind parameter is not 1 (user defined recipes because the recipes that can be synced to the Z does not exceed 20).

##### Request

`POST https://www.picobrew.com/Vendors/input.cshtml?ctl=RecipeRefListController&token=<token> HTTP/1.1`

Query String:

* token:  The unique identifier for the Z

JSON Request Body:

* Kind:  List search type 
    * 1 = user defined recipes
    * 2 = appears to be unbrewed paks I've purchased?
    * 3 = browse/search entire zpak list, search provides SearchString in request.
    * 4 = zpak id search. Other values have not been observed yet.
* MaxCount:  The requested limit for the number of recipes to be returned.  Appears to always be 20.
* Offset:  Suspected to be the number of recipes to skip for pagination purposes. Seen in 'Kind:3' and 'Kind:4'. Pagination does not appear to be supported in 'Kind:1' 
* SearchString: An optional string for search. May only be supported in Kind:4

Example:

```
{
    "Kind": 1,
    "MaxCount": 20,
    "Offset": 0,
}
```

##### Response

* Recipes: An array of recipes where each recipe is:
    * Abv:  Appears to always be -1, except for Kind:2 where values have been observed.
    * ID: Unique numeric identifier for the recipe.
    * Ibu:  Appears to always be -1, except for Kind:2 where values have been observed.
    * Name:  The name of the recipe. Further testing needed to determine max length.
    * Kind:  Appears to be zero, regardless of whether it is a beer, coffee, or sous vide recipe. Appears as Recipies.Kind = 1 in Kind:2 (Pak search)
    * Uri:  Appears to always be null.
* SearchString:  Same as sent in Request ( see Kind:3 )
* TotalResults:  A number between 0 and 20.  It's unclear how the Picobrew will behave if the number is greater than 20.  My best guess is that a second Fetch Recipe query will be sent with a value of 20 for the offset and so on.

Example:

```
{
    "Kind": 1,
    "MaxCount": 0,
    "Offset": 0,
    "Recipes": [
        {
            "Abv": -1,
            "ID": 150xxx,
            "Ibu": -1,
            "Kind": 0,
            "Name": "All Good Things",
            "Uri": null
        },
        {
            "Abv": -1,
            "ID": 150xxy,
            "Ibu": -1,
            "Kind": 0,
            "Name": "New Coffee",
            "Uri": null
        },
        {
            "Abv": -1,
            "ID": 150xxz,
            "Ibu": -1,
            "Kind": 0,
            "Name": "Test SV Recipe",
            "Uri": null
        }
    ],
    "SearchString": null,
    "TotalResults": 3
}
```

##### Request (ZPak Search)

Example of a ZPak Search for "star"
```
{
    "Kind": 3,
    "MaxCount": 20,
    "Offset": 0,
    "SearchString": "star"
}

```

##### Response (ZPak Search)

```
{
    "Kind": 3,
    "Offset": 0,
    "SearchString": "star",
    "MaxCount": 0,
    "TotalResults": 1,
    "Recipes": [
        {
            "ID": 139605,
            "Name": "Stargazer Z Kit",
            "Kind": 0,
            "Uri": null,
            "Abv": -1,
            "Ibu": -1
        }
    ]
}
```

##### Request (ZPak Browse, Page 2 Page 2 Results)

```
{
    "Kind": 3,
    "MaxCount": 20,
    "Offset": 20
}
```

###### Response (ZPak Browse, Page 2 Results)
```
{
    "Kind": 3,
    "Offset": 20,
    "SearchString": null,
    "MaxCount": 0,
    "TotalResults": 27,
    "Recipes": [
        {
            "ID": 139601,
            "Name": "Sandy Beaches Z Kit",
            "Kind": 0,
            "Uri": null,
            "Abv": -1,
            "Ibu": -1
        },
        {
            "ID": 139598,
            "Name": "Shankill Stout Z Kit",
            "Kind": 0,
            "Uri": null,
            "Abv": -1,
            "Ibu": -1
        },
        {
            "ID": 140586,
            "Name": "Smokey Lonesome ZKit",
            "Kind": 0,
            "Uri": null,
            "Abv": -1,
            "Ibu": -1
        },
        {
            "ID": 139605,
            "Name": "Stargazer Z Kit",
            "Kind": 0,
            "Uri": null,
            "Abv": -1,
            "Ibu": -1
        },
        {
            "ID": 141309,
            "Name": "Stefanator Doppel",
            "Kind": 0,
            "Uri": null,
            "Abv": -1,
            "Ibu": -1
        },
        {
            "ID": 141320,
            "Name": "The Citra Galaxy",
            "Kind": 0,
            "Uri": null,
            "Abv": -1,
            "Ibu": -1
        },
        {
            "ID": 140579,
            "Name": "Tweaties Z Kit",
            "Kind": 0,
            "Uri": null,
            "Abv": -1,
            "Ibu": -1
        }
    ]
}
```

## Fetch Recipe Details

After a recipe is selected, the Z must get all of the steps for the recipe.  This request is used to fetch that information.

##### Request

`GET https://www.picobrew.com/Vendors/input.cshtml?type=Recipe&token=<token>&id=<recipe_id> HTTP/1.1`

Query String:

* token:  The unique identifier for the Z
* recipe_id:  The unique identifier of the recipe, as specified in the results of a fetch recipe list request.

##### Response

* ID: The unique recipe identifier
* Name:  The name of the recipe. Appears to truncate to a length < 20
* StartWater:  The amount of water (in liters) that the keg should be filled with at the start of the brewing session.
* Steps:  An array of step objects.
    * Drain:  The amount of time (in minutes) that a drain cycle should be run for at the completion of the step.  This time is not included in the amount of time specified in the Time field.
    * Location:  The target for input fluid.
        * 0 = Pass Through
        * 1 = Mash
        * 2 = Adjunct 1
        * 3 = Adjunct 2
        * 4 = Adjunct 3
        * 5 = Adjunct 4
        * 6 = Special value to indicate that the machine should pause for the user.
    * Name:  The name of the step, this string appears to truncate to a maximum length < 20
    * Temp:  The target temperature for the step, in degrees fahrenheit.
    * Time:  The amount of time to spend at the target temperature during this step.
* TypeCode:  The type of recipe, possible values:
    * `Beer` (self explanatory)
    * `Coffee` (self explanatory)
    * `Other`:  This value was observed in a sous vide recipe

Example:

```
{
    "ID": 150xxx,
    "Name": "All Good Things",
    "StartWater": 13.1,
    "Steps": [
        {
            "Drain": 0,
            "Location": 0,
            "Name": "Heat to Dough In",
            "Temp": 100,
            "Time": 0
        },
        {
            "Drain": 8,
            "Location": 1,
            "Name": "Dough In",
            "Temp": 100,
            "Time": 2
        },
        {
            "Drain": 0,
            "Location": 0,
            "Name": "Heat to Mash 1",
            "Temp": 105,
            "Time": 0
        },
        {
            "Drain": 8,
            "Location": 1,
            "Name": "Mash 1",
            "Temp": 105,
            "Time": 2
        },
        {
            "Drain": 0,
            "Location": 0,
            "Name": "Heat to Mash 2",
            "Temp": 110,
            "Time": 0
        },
        {
            "Drain": 8,
            "Location": 1,
            "Name": "Mash 2",
            "Temp": 110,
            "Time": 2
        },
        {
            "Drain": 0,
            "Location": 0,
            "Name": "Heat to Mash Out",
            "Temp": 115,
            "Time": 0
        },
        {
            "Drain": 8,
            "Location": 1,
            "Name": "Mash Out",
            "Temp": 115,
            "Time": 2
        },
        {
            "Drain": 0,
            "Location": 2,
            "Name": "Heat to Boil",
            "Temp": 190,
            "Time": 0
        },
        {
            "Drain": 0,
            "Location": 2,
            "Name": "Boil Adjunct 1",
            "Temp": 190,
            "Time": 3
        },
        {
            "Drain": 0,
            "Location": 3,
            "Name": "Boil Adjunct 2",
            "Temp": 190,
            "Time": 1
        },
        {
            "Drain": 0,
            "Location": 4,
            "Name": "Boil Adjunct 3",
            "Temp": 190,
            "Time": 1
        },
        {
            "Drain": 0,
            "Location": 0,
            "Name": "Cool to Whirlpoo...",
            "Temp": 180,
            "Time": 0
        },
        {
            "Drain": 5,
            "Location": 5,
            "Name": "Whirlpool Step",
            "Temp": 180,
            "Time": 5
        },
        {
            "Drain": 0,
            "Location": 6,
            "Name": "Connect Chiller",
            "Temp": 61,
            "Time": 0
        },
        {
            "Drain": 10,
            "Location": 0,
            "Name": "Chill",
            "Temp": 61,
            "Time": 10
        }
    ],
    "TypeCode": "Beer"
}
```


## Begin/Complete a Program Session

##### Request

`PUT https://www.picobrew.com/Vendors/input.cshtml?type=ZSession&token=<token>&id=<session_id> HTTP/1.1`

When beginning a new program session, the session_id is not included in the query string.  This value is present when completing a program session and is the only observed request difference between a begin session and complete session request.

JSON Request Body:

* DurationSec
* FirmwareVersion
* GroupSession
* MaxTemp
* MaxTempAddedSec
* Name
* PressurePa
* ProgramParams
    * Abv:  Appears to always be -1
    * Duration
    * Ibu
    * Intensity
    * Temperature:   
    * Water: The amount of water (in liters) for the program.
* RecipeID: The unique recipe ID for the recipe.  For custom recipes and built-in programs, the values 0 and -1 have been observed.
* SessionType: All possible values are not know, the following values have been observed.
    * 0 - Rinse
    * 1 - Clean
    * 2 - Drain
    * 3 - Rack Beer
    * 4 - Circulate
    * 5 - Sous Vide
    * 6 - Beer
    * 12 - Coffee
    * 13 - Chill
* ZProgramId: All possible values are not know, the following values have been observed.
    * 1 - Rinse
    * 2 - Drain
    * 3 - Rack Beer
    * 4 - Circulate
    * 6 - Sous Vide
    * 12 - Clean
    * 24 - Beer or Coffee
    * 27 - Chill

Example 1:  (A Beer Recipe)

```
{
    "DurationSec": 11251,
    "FirmwareVersion": "0.0.116",
    "GroupSession": true,
    "MaxTemp": 98.22592515,
    "MaxTempAddedSec": 0,
    "Name": "All Good Things",
    "PressurePa": 101975.6172,
    "ProgramParams": {
        "Abv": -1,
        "Duration": 0,
        "Ibu": -1,
        "Intensity": 0,
        "Temperature": 0,
        "Water": 13.1
    },
    "RecipeID": 150xxx,
    "SessionType": 6,
    "ZProgramId": 24
}
```

Example 2:  (A Custom Coffee Recipe)

```
{
    "DurationSec": 1983,
    "FirmwareVersion": "0.0.116",
    "GroupSession": true,
    "MaxTemp": 0,
    "MaxTempAddedSec": 0,
    "Name": "Custom Coffee 4.0L (7 min @109F)",
    "PressurePa": 101749.3594,
    "ProgramParams": {
        "Abv": -1,
        "Duration": 420,
        "Ibu": -1,
        "Intensity": 7,
        "Temperature": 109.4,
        "Water": 4
    },
    "RecipeID": 0,
    "SessionType": 12,
    "ZProgramId": 24
}
```

Example 3:  (A Custom Sous Vide Recipe)

```
{
    "DurationSec": 9835,
    "FirmwareVersion": "0.0.116",
    "GroupSession": true,
    "MaxTemp": 98.14043988,
    "MaxTempAddedSec": 0,
    "Name": "SOUSVIDE",
    "PressurePa": 101729.4375,
    "ProgramParams": {
        "Abv": -1,
        "Duration": 420,
        "Ibu": -1,
        "Intensity": 7,
        "Temperature": 109.4,
        "Water": 4
    },
    "RecipeID": -1,
    "SessionType": 5,
    "ZProgramId": 6
}
```

##### Response

Note:  The PressurePa does not appear to be echoed back in this response.  A Pressure field is present but does not appear to have a value other than zero.

* Active: If true, indicates the session is active, if false, indicates the session has been completed.
* CityLat: The user-configured latitude of the city in which they reside.
* CityLng: The user-configured longtitude of the city in which they reside.
* ClosingDate:  If the session has completed, a UTC time stamp otherwise null.
* CreationDate:  UTC time stamp for the start of the session.
* Deleted:  Appears to always be false when communicating with the Z.
* GUID:  Uncertain of this value, but it appears to be unique per brewing session.
* ID:  The unique ID for this brew session.
* LastLogID:  The unique identifier for the final session update for this brewing session.
* Lat: The user-configured latitude of their address.
* Lng: The user-configured longtitude of their address.
* Notes:  Generally appears to be null.
* Pressure:  Appears to always be 0.
* ProfileID:  The numeric ID for the user
* RecipeGuid:  
* SecondsRemaining:  For beginning of sessions, the length of the session in seconds.  For completion of sessions, null.
* SessionLogs:  This appears to be an empty array.
* StillUID:  Unknown, likely related to PicoStill.
* StillVer:  Unknown, likely related to PicoStill.
* ZSeriesID:  The numeric ID for the Z.  This differs from the identifying token.

Example: (Begin Session)

```
{
    "Active": false,
    "CityLat": xx.xxxxxx,
    "CityLng": -yyy.yyyyyy,
    "ClosingDate": "2020-05-04T19:54:58.74",
    "CreationDate": "2020-05-04T19:46:04.153",
    "Deleted": false,
    "DurationSec": 578,
    "FirmwareVersion": "0.0.116",
    "GUID": "<all upper case guid>",
    "GroupSession": false,
    "ID": <session id>,
    "LastLogID": 11407561,
    "Lat": xx.xxxxx,
    "Lng": -yyy.yyyyyy,
    "MaxTemp": 98.24455443,
    "MaxTempAddedSec": 0,
    "Name": "RINSE",
    "Notes": null,
    "Pressure": 0,
    "ProfileID": zzzz,
    "ProgramParams": {
        "Abv": null,
        "Duration": 0.0,
        "Ibu": null,
        "Intensity": 0.0,
        "Temperature": 0.0,
        "Water": 0.0
    },
    "RecipeGuid": null,
    "RecipeID": null,
    "SecondsRemaining": 0,
    "SessionLogs": [],
    "SessionType": 0,
    "StillUID": null,
    "StillVer": null,
    "ZProgramId": 1,
    "ZSeriesID": wwww
}
```

## Provide a Program Session Update

##### Request

`POST https://www.picobrew.com/Vendors/input.cshtml?type=ZSessionLog&token=<token> HTTP/1.1`

JSON Request Body:

* AmbientTemp:  The value, in Celsius, observed by the ambient temperature sensor.
* DrainPumpOn:  1 if the drain pump is on, otherwise 0.  (This is the pump TO the keg.)
* DrainTemp:  The value, in Celsius, observed by the drain pump temperature sensor.
* ErrorCode:  
* KegPumpOn:  1 if the keg pump is on, otherwise 0.  (This is the pump FROM the keg.)
* PauseReason: Appears to be 1 for Preparing machine and pre-brew tests, other values are unknown.
* SecondsRemaining:  The number of seconds remaining for the current program session.
* StepName:  The name of the current step.
* TargetTemp:  The target temperature of the current step.
* ThermoBlockTemp:  The temperature observed by the heating sensor, sometimes referred to as the hex temperature.
* ValvePosition:  The current position in the valve, with values matching the locations.  (Need to confirm.)
* WortTemp:  The temperature observed by the wort sensor.
* ZSessionId:  The unique session ID for the current program.
* netRecv:  Unknown.
* netSend:  Unknown.
* netWait:  Unknown.
* rssi:  Unknown.

Example:

```
{
    "AmbientTemp": 24.334857,
    "DrainPumpOn": 0,
    "DrainTemp": 23.18313793,
    "ErrorCode": 0,
    "KegPumpOn": 1,
    "PauseReason": 0,
    "SecondsRemaining": 289,
    "StepName": "Rinsing",
    "TargetTemp": 0,
    "ThermoBlockTemp": 23.70420299,
    "ValvePosition": 4,
    "WortTemp": 23.33230584,
    "ZSessionID": <session_id>,
    "netRecv": 84,
    "netSend": 1326,
    "netWait": 1770,
    "rssi": -26
}
```

##### Response

* ID:  The unique identifier for this session update.
* StillSessionLog:  Unknown
* StillSessionLogID:  Unknown

Example:

```
{
    "AmbientTemp": 24.334857,
    "DrainPumpOn": false,
    "DrainTemp": 23.18313793,
    "ErrorCode": 0,
    "ID": 11407374,
    "KegPumpOn": true,
    "LogDate": "2020-05-04T19:50:36.74",
    "PauseReason": 0,
    "SecondsRemaining": null,
    "StepName": "Rinsing",
    "StillSessionLog": null,
    "StillSessionLogID": null,
    "TargetTemp": 0.0,
    "ThermoBlockTemp": 23.70420299,
    "ValvePosition": 4.0,
    "WortTemp": 23.33230584,
    "ZSessionID": <session_id>,
    "netRecv": 84,
    "netSend": 1326,
    "netWait": 1770,
    "rssi": -26
}
```

### Resume a Session

This request is made by the Z when the user has indicated that they wish to resume the program that was running at the time the Z lost power.  So far, this is only documented for brewing recipes.

##### Request

`GET https://www.picobrew.com/Vendors/input.cshtml?type=ResumableSession&token=<token>&id=<session_id> HTTP/1.1`

* token:  The unique identifier for the Z
* session_id: The unique session ID for the session to be resumed, provided by ResumableSessionID in the initial boot synchronization.

##### Response

The response includes the remaining recipe steps for the PicoBrew and is based upon the last session update.  All completed steps are removed from the recipe and the time for the current step is reduced to the amount of time remaining for that step as of the last session update.

* Recipe:  An array of remaining recipe steps, with entries as defined in fetch recipe details request.
* SessionID:  The unique session ID for the brewing session.
* SessionType:  The session type for the session, as defined in begin program session.
* ZPicoRecipe:  Purpose unknown, generally appears to be null.

Example:

```
{
    "Recipe": {
        "ID": 150xxx,
        "Name": "All Good Things",
        "StartWater": 13.1,
        "Steps": [
            {
                "Drain": 5,
                "Location": 5,
                "Name": "Whirlpool Step",
                "Temp": 180,
                "Time": 1
            },
            {
                "Drain": 0,
                "Location": 6,
                "Name": "Connect Chiller",
                "Temp": 61,
                "Time": 0
            },
            {
                "Drain": 10,
                "Location": 0,
                "Name": "Chill",
                "Temp": 61,
                "Time": 10
            }
        ],
        "TypeCode": "Beer"
    },
    "SessionID": 60xxx,
    "SessionType": 6,
    "ZPicoRecipe": null
}
```
