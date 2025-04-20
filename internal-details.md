# Introduction 
This guide explains how to modify official Debezium connectors to work with a self-hosted Confluent Platform. The example used here is the `debezium-connector-mariadb`, but the steps should apply to other Debezium connectors as well.

### Step 1: Compare the differences between the Debezium connector on Confluent Hub and the official distribution
Since the `debezium-connector-mariadb` is not available on Confluent Hub, the `debezium-connector-mysql` is used for comparison.

1a. Download the official connector from [Debezium's official website](https://debezium.io/documentation/reference/stable/install.html).  
1b. Download the Confluent Hub connector from [Confluent Hub](https://www.confluent.io/hub/debezium/debezium-connector-mysql).

**Tip:** Edit the download links from the official source to match the version of the connector you downloaded from Confluent Hub.

You will notice that the official connector has all files (of different types) in the base folder, while the Confluent Hub connector organizes files into separate folders for JARs and documentation.

### Step 2: Download and reorganize files for the `debezium-connector-mariadb`
Download the official MariaDB connector from the [Debezium download page](https://debezium.io/documentation/reference/stable/install.html) and reorganize the files based on the observations from Step 1.

2a. Place all `.jar` files in the `lib` folder.  
2b. Place all `.txt` and `.md` files in the `docs` folder.  
2c. Copy the `assets` and `etc` folders from the MySQL connector to your MariaDB connector.  
2d. Create a `manifest.json` file for your connector. You can copy this file from the MySQL connector and update the relevant fields, such as `name`, `version`, and `title`.

### Step 3: Package your connector and compute its checksum
3a. Package your connector into a `.zip` file, similar to the format used for connectors downloaded from Confluent Hub.  
3b. Compute the SHA-512 checksum of your `.zip` file using any CLI or online tool.  
3c. Host your connector `.zip` file at a location accessible from your network via HTTP. (For example, you can host it on GitHub.)

---

# Debugging Installation Issues
This section is specifically relevant if you are using Confluent's Kubernetes Operator, where the init container is responsible for downloading and installing your plugin.

You can download and run the init container's Docker image separately to inspect its files and understand how it installs the connector. In the version tested, the `/opt/startup.sh` script in the image invokes the `/opt/generate_install_plugin_script.py` script to download and install the plugin.

The `generate_install_plugin_script.py` script is essentially a wrapper around the `confluent-hub` CLI and executes the `confluent-hub install` command to install the plugin.

### Commands to pull and run the init container in a Docker environment:
```bash
docker pull confluentinc/confluent-init-container:2.7.0
docker run -it --entrypoint sh confluentinc/confluent-init-container:2.7.0
```

Contents of the `generate_install_plugin_script.py` script:
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
