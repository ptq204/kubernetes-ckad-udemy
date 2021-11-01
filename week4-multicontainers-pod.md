## Multicontainer Pod

There are 3 common patterns:  

`Side car`: for example, deploy a log service along side with our service to collect logs and send those to the central log server.  

`Adapter`: in the example log service above, each log service may collect logs under different format. Therefore, we need an adapter to convert those logs format into one unique format that will can be processed by the central log server.  

`Ambassador`: 