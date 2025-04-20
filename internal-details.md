# Introduction 
This guide details how to modify official debezium connectors so that they'll work with self-hosted confluent platform. I'll take the example of debezium-connector-mariadb, but this should also apply to other debezium connectors.

### Step 1: Compare the differences between debezium connector on confluent hub, and the one present on official distribution.
Since the debezium-connector-mariadb isn't available on confluent hub, I used debezium-connector-mysql for this comparison.

1a. Download the official connector from https://debezium.io/documentation/reference/stable/install.html
1b. Download the confluent hub connector from https://www.confluent.io/hub/debezium/debezium-connector-mysql

Tip: You can edit the download links from official source to point to the same version of connector as you've downloaded from confluent-hub.

Here, you'll notice that the official connector has all the files (of different types) in the base folder, while the confluent-hub connector has different folders for jars and docs.

### Step 2: Download and reorganize files in the debezium-connector-mariadb
Download the official mariadb connector from the [debezium download page](https://debezium.io/documentation/reference/stable/install.html), and reorganize files according to the observations in previous step.

2a. Put all the *.jar* files in the *lib* folder
2b. Put all the *.txt* and *.md* files in the *docs* folder
2c. Copy and paste the *assets* and *etc* folders from the other connector to your own connector
2d. Create a manifest.json file for your connector. You can copy the file from the mysql connector and replace the important fields such as name, version and title.

### Step 3: Package your connector and compute checksum
3a. Package your connector in a *.zip* file, just like the one downloaded from confluent hub.
3b. Compute sha512 checksum of your zip file, using any CLI/online tool.
3c. Host your connector zip file at a location so that it's accessible from your network using http. (We've hosted it on github :)


# Debugging installation issues
This part is specifically applicable if you're using the confluent's kubernetes operator, so that the init container is the one responsible for downloading and installing your plugin.

You can separately download the init container's docker image and run it, to inspect the files under it and how it installs the connector. In the version I tested on, there's a `/opt/startup.sh` script present in that image, which in-turn invokes the `/opt/generate_install_plugin_script.py` script to download and install the plugin.

The `generate_install_plugin_script.py` is really a wrapper over the `confluent-hub` CLI and ends up running the `confluent-hub install` command to actually install the plugin.

Here are commands to pull and run the init container separately in a docker environment:
```bash
docker pull confluentinc/confluent-init-container:2.7.0
docker run -it --entrypoint sh confluentinc/confluent-init-container:2.7.0
```

Here are the contents of `generate_install_plugin_script.py` script:
```python
#!/usr/bin/env python

import sys
import json
import os

ZIP_EXTENSION = ".zip"

plugins_json_file = sys.argv[1]
output_file = sys.argv[2]

def get_file_name(path, name, type):
    return path + '/' + name + type

with open(plugins_json_file) as pf:
    plugins_object = json.load(pf)

with open(output_file, 'w+') as of:
    of.write('#!/usr/bin/env sh \n')

    confluentHubPlugins = plugins_object['ConfluentHubPlugins']
    if confluentHubPlugins is not None:
        for plugin in confluentHubPlugins:
            of.write(f"confluent-hub install {plugin['Name']} --no-prompt --worker-configs /dev/null --component-dir /mnt/plugins {plugin['ExtraArgs']} \n")

    urlPlugins = plugins_object['RemoteURLPlugins']
    if urlPlugins is not None:
        for plugin in urlPlugins:
            filename = get_file_name("/mnt/plugins", plugin['Name'], ZIP_EXTENSION)
            # download file. set ""--max-redirect 0" for CVE-2021-31879
            of.write(f"wget --max-redirect 0 {plugin['ArchivePath']} -O {filename} -P /mnt/plugins {plugin['WgetExtraArgs']} \n")

            # verify checksum
            sha512filename = get_file_name("/mnt/plugins", plugin['Name'], '.sha512')
            of.write(f"echo {plugin['Checksum']} {filename} > {sha512filename} \n")
            of.write(f"sha512sum --check {sha512filename} \n")
            of.write(f"rm -f {sha512filename} \n")

            # install with confluent-hub
            of.write(f"confluent-hub install {filename} --no-prompt --worker-configs /dev/null --component-dir /mnt/plugins {plugin['ExtraArgs']} \n")
            of.write(f"rm -vf {filename} \n")
os.chmod(output_file, 0o755)
```
