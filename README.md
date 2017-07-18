# cfgbak

### About
Automates the process of backing up network device configurations. This tool reads device information from a
configuration file, connects to each device via SSH, and initiates an file upload back to a built-in TFTP server.

Device classes are configured with an expect macro written in Javascript which is executed at runtime against an SSH
session. Support for additional device types can easily be added by writing a expect macro.

The sample config has example expect macros for Cisco ISRs, HP switches, and the Ubiquiti EdgeRouter.

### Quick Example
```
# Aquire & build the tool
go get .
go build

# Copy the example configuration
cp config.example.conf config.conf

# Edit as needed
vim config.conf

# Run the backup
./cfgbak --config config.conf
```

## Configuration


### Preferences
```hcl
preferences {
    // The destination directory for all configurations
    backup_dir = "./config-backups/"
    
    // The external IP address of our computer. This is the address devices will use to connect
    // to our computer and upload the configs via TFTP 
    host_ip = "192.168.1.10"
}
```

### Device Classes
The `device_class` block defines a device class that can be attached to multiple devices. Inside the
`device_class` block you can specify multiple `backup_target`s. `cfgback` will open an SSH session and evaluate the
defined expect `macro` for each `backup_target` defined. The `macro` is executed in a JavaScript VM at runtime. 
```hcl
// Sample device class for backing up Cisco ISR routers
device_class "cisco_isr" {
    
    // Specifiy a backup_target for the 'startup-config' file
    backup_target "startup_config" {
        
        // This is the expect macro that gets executed at runtime in the SSH session to trigger the
        // TFTP upload back to our built-in server.
        macro = <<-MACRO
            expect("#")
            sendLine("copy startup-config tftp://" + ctx.TFTPHost + "/" + ctx.TFTPFilename)
            expect("["+ctx.TFTPHost+"]?")
            sendLine("")
            expect("["+ctx.TFTPFilename+"]?")
            sendLine("")
            expect("#")
        MACRO
    }
}
```

### Auth Providers
The `auth_provider` block defines an authentication provider that will provide device credentials at
runtime. There are currently two supported `auto_provider` types available.   

#### Static Provider
The `static` `auto_provider` uses credentials that are stored plaintext in the configuration file. You can specify
additional options using the `attributes` block. Fields defined in the `attributes` block can be retrieved at runtime in
a JavaScript macro using the `getAuthAttr(field_name)` function.
```hcl
auth_provider "static" "my_auth" {
    auth "cisco_router_auth" {
        username = "john_doe"
        password = "secret"
    }
    
    // Attributes can be queried at runtime by an expect macro for additional authentication information. This is a
    // convienient way to store enable passwords.
    attributes (
        my_optional_attribute = "something"
    )
}
```
#### KeePass Provider
The `keepass` `auto_provider` uses credentials that are stored in a KeePass database. Currently the `unlock_credential`
is still stored in plaintext though.
```hcl
auth_provider "keepass" "my_auth_db" {
    db_path = "./my_secrets.kdbx"
    unlock_credential = "secretpassword"
}
```

### Devices
The `device` block defines a specific that we want to backup. We specify a `device_class`, in this case
`cisco_isr`, which associates the `device_class`'s `backup_target`'s with the `device`
```hcl
device "cisco_isr" "test_router_1" {
    address = "192.168.1.1:22"
    auth = "my_auth:cisco_router_auth"
}
```

### Includes
The `include` block specifies a configuration file to include. 
```hcl
include "file_to_include.conf" {}
```