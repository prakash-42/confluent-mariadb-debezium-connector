# Introduction 
This guide explains how to modify official Debezium connectors to work with a self-hosted Confluent Platform. The example used here is the `debezium-connector-mariadb`, but the steps should apply to other Debezium connectors as well.

## Step 1: Compare the differences between the Debezium connector on Confluent Hub and the official distribution
Since the `debezium-connector-mariadb` is not available on Confluent Hub, the `debezium-connector-mysql` is used for comparison.

1. **Download the official connector** from [Debezium's official website](https://debezium.io/documentation/reference/stable/install.html).  
2. **Download the Confluent Hub connector** from [Confluent Hub](https://www.confluent.io/hub/debezium/debezium-connector-mysql).

**Tip:** Edit the download links from the official source to match the version of the connector you downloaded from Confluent Hub.

### Observations:
- The official connector places all files (of various types) in the base folder.
- The Confluent Hub connector organizes files into separate folders for JARs and documentation.

## Step 2: Download and reorganize files for the `debezium-connector-mariadb`
1. **Download the official MariaDB connector** from the [Debezium download page](https://debezium.io/documentation/reference/stable/install.html).  
2. **Reorganize the files** based on the observations from Step 1:
   - Place all `.jar` files in the `lib` folder.  
   - Place all `.txt` and `.md` files in the `docs` folder.  
   - Copy the `assets` and `etc` folders from the MySQL connector to your MariaDB connector.  
   - Create a `manifest.json` file for your connector. You can copy this file from the MySQL connector and update the relevant fields, such as `name`, `version`, and `title`.

## Step 3: Package your connector and compute its checksum
1. **Package your connector** into a `.zip` file, similar to the format used for connectors downloaded from Confluent Hub.  
2. **Compute the SHA-512 checksum** of your `.zip` file using any CLI or online tool.  
3. **Host your connector** `.zip` file at a location accessible via HTTP (e.g., GitHub).

---

# Debugging Installation Issues
This section is specifically relevant if you are using Confluent's Kubernetes Operator, where the init container is responsible for downloading and installing your plugin.

### Steps to Debug:
1. **Download and run the init container's Docker image** to inspect its files and understand how it installs the connector:
   ```bash
   docker pull confluentinc/confluent-init-container:2.7.0
   docker run -it --entrypoint sh confluentinc/confluent-init-container:2.7.0
   ```
2. Inspect the `/opt/startup.sh` script in the image. This script invokes the `/opt/generate_install_plugin_script.py` script to download and install the plugin.

### About the `generate_install_plugin_script.py` Script:
The `generate_install_plugin_script.py` script is a wrapper around the `confluent-hub` CLI. It executes the `confluent-hub` install command to install the plugin.

**Script Contents**
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
