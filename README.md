# Home Assistant - History to InfluxDB

## Differences to the orginal
 - SQLite is enabled by default
 - source datebase type can be set by a parameter (mariasql/mysql/sqlit)
 - runs independently in a docker environment - cleaner uninstall
 - data loose beween import and using influx depends only on HA restart and db copy time 
   - it is not recommended to copy the SQLite db without stoping HA
 - my setup:
   - Pi 4 + M2 SSD
   - copy 2.5 GB SQLite
   - 1 minute HA shutdown

## Quick Steps
  - prepare SSH to HA
  - prepare InfluxDB to use by HA at the next HA startup (install; add db + user; add HA settings)
  - login to SSH and clone repo
  - prepare scripts - stop HA - copy DB - start HA
    - for now HA write also to InfluxDB
  - run script
    - the old data is also in the InfluxDB
  - cleanup
 
## Important

As this script is used once and then (at least in theory) never again, I will
not be able to provide support or testing. Please check the forums as well as
any forks of the repo for potential updates by the community.

Quality of the script is also disputable given that it is a one-off. Use of
MySQL/MariaDB and SQLite is hard-coded.

Use at your own risk. (Backups recommended)

## Introduction

Home Assistant's recorder component allows to store historical data in a database.
Database access is handled by SQLAlchemy, with the default database in SQLite.
MySQL/MariaDB is also quite popular and so is PostgreSQL.

However, if one wants to store a lot of data over a long period of time, neither
of these options gives the best performance. Instead, a dedicated time-series
keeping database format like InfluxDB allows best retrieval and storage of the
data.

However, if you only figure this out after already having assembled a huge amount
of historical data, there is no option to migrate your data.

This is an attempt to do exactly that. It is a one-off migration of data to
InfluxDB. Afterwards you should setup the InfluxDB integration to directly store
data to InfluxDB (and only keep a couple of days to few weeks at most in the
traditional database for the logbook and history components).

References:
- https://www.home-assistant.io/integrations/recorder/
- https://www.home-assistant.io/integrations/influxdb/

## Caveats

This script is rather simple and limited to my use-case. As it is a one-off
and I do not have other setups readily available, I limited it to the specific
task at hand. However, it should be easily adaptable.

Namely, this handles MySQL / MariaDB / SQLite only. Adding PostgreSQL etc could
be done trivially, I believe.

## Setup

In order to not duplicate logic, the script uses the InfluxDB component of
Home Assistant directly.

Tested on 
- HomeAssistant OS with docker python image

## Preparation
1. Enable SSH access to HomeAssistant
   - Supervisor -> install SSH & Web Terminal
   - configure addon
   - disable Protecton mode (allow docker access via ssh)
   - start addon
2. Install InfluxDB
   - Supervisor -> install InfluxDB
   - configure addon
   - start addon
   - configure user + database
   - configure configuration.yaml + influxdb.yaml (without !secret)
   - Use and edit the provided influxdb.yaml example OR copy your InfluxDB 
     configuration from Home Assistant to influxdb.yaml.
     This should be the file that you include via `influxdb: !include influx.yaml`
     in your installation (i.e. it does not start with `influxdb:\n`!).
     It must not include any !secret statements but rather the token (for v2)
     or user/password (for v1) explicitly

## Installation
1. login via ssh twice (**host** session / **container** session)
2. **container** session
   - `docker run -it --network host --name python python bin/bash`
   - `git clone https://github.com/chriskuba/homeassistant2influxdb.git migrate2influxdb`
   - `cd migrate2influxdb`
   - `git clone https://github.com/home-assistant/core.git home-assistant-core`
   - `python3 -m venv .venv`
   - `. .venv/bin/activate`
   - `pip install -r requirements.txt`
   - `pip install -r home-assistant-core/requirements.txt`
3. **host** session
   - if you will use the influxdb.yaml of the HA installation; otherweise  edit the template within the **container** session
      - `docker cp /config/influxdb.yaml python:/migrate2influxdb`
   - if you use the default sqlite db of HA
      - `ha core stop`
      - `docker cp /config/home-assistant_v2.db python:/migrate2influxdb`
      - `ha core start`

## Run script:
1. **container** session
   - `python homeassistant2influxdb.py ...` (see -h for options to specify your MariaDB/MySQL credentials - default is SQLite)

## Cleanup:
1. **container** session
   - `exit` (the container)
   - `exit` (the ssh shell)
2. **host** session
   - `docker container rm python`
   - `docker image rm python`
3. remove the SSH & Web Terminal addon from HomeAssistant or enable the Protection mode
4. additionally use secrets in influxdb.yaml now
5. maybe runcate / reconfgure the HA record database
