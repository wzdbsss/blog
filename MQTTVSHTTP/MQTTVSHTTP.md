# MQTT vs HTTP

||MQTT|HTPP|
|-|-|-|
|Design orientation|Data centric|Document centric|
|Pattern|Publish/subscribe|Request/response|
|Complexity|Simple|More complex|
|Message size|Small, with a compact binary header just two bytes in size|Larger, partly because status detail is text-based|
|Service levels|Three quality of service settings|All messages get the same level of service|
|Extra libraries|Libraries for C (30 KB) and Java (100 KB)|Depends on the application (JSON, XML), but typically not small|
|Data distribution|Supports 1 to zero, 1 to 1, and 1 to n|1 to 1 only|
