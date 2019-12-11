# G1ANT.Robot.API

As a developer, we would like to have remote access to G1ANT.Robot by built-in WebServer and REST:

1. For remote control
2. For remote management
3. For remote administration
4. For remote development (spread the processes)

Everything from that is possible with G1ANT.Robot.API feature. Everybody can build
G1ANT.Orchestrator (for Azure, for our own or connector to any different system), by using this API.

![Orchestrator](orchestrator.jpg)

## Settings

First of all, we have to find G1ANT.Robot.config file which is stored in user Documents\G1ANT.Robot folder.
We have to put the last <WebServer> section:

```XML
<?xml version="1.0"?>
<Settings xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://g1ant.com/XmlSchema/v3/Settings">
 ...
 <WebServer>
  <Enabled>true</Enabled>
  <Port>1234</Port>
    <Tokens>
      <Token Name="Chris" Value="9CE867D195B02E" />
    </Tokens>
  </WebServer>
</Settings>
```

Whereas

Name | Description
-----|------------
**Enabled** | API can be enabled (true) or no (false)
**Port** | On which port our API should be available? 1234 is ok.
**Tokens** | List of tokens for authorisation. Value is the key. Name is for our information only.

## Authporisation

Each query should have two parameters to authorise transaction: token from settings file and serial of our program. 
Let's execute our first query. For example, let's check all available addons on the robot side:

```
```