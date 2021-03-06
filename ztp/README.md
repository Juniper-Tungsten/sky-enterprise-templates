# ZTP Templates

## Getting Started with ZTP XML Templates

### Introduction

To get started with Sky Enterprise you can take a look at this Juniper guide which outlines some core areas of Sky Enterprise and ZTP Templates:

https://www.juniper.net/documentation/en_US/sky-enterprise/topics/topic-map/getting-started-guide.html#id-creating-a-ztp-template

### Initial Template Definition.

Get a starting point using the output from 'show configuration | display xml' this will also then need the following header and footer:

``` xml
<device xmlns="http://juniper.net/zerotouch-bootstrap-server"> <unique-id>{{ serial }}</unique-id>
    <configuration>
        <config> 
                ...
        </config>
    </configuration>
</device>
```

The factory default configuration will contain the system autoinstallation line which needs to be removed:

``` xml
    <autoinstallation delete="delete"/>
```

Then any configuration that clashes with the factory default needs to be deleted, so for example to set the IP address of irb.0 and overwrite the default config:

``` xml
          <interface delete="delete">
            <name>irb</name>
          </interface>
          <interface>
            <name>irb</name>
            <unit>
              <name>0</name>
              <family>
                 <inet>
                  <address>
                    <name>{{ irb_address }}</name>
                  </address>
                </inet>
              </family>
            </unit>
          </interface>
```

Remove any unnecessary default configuration using delete syntax. For example the default dhcp server configuration:

``` xml
      <services>
      <dhcp-local-server delete="delete">
      </services>
      <access delete="delete">
```

To complete the template, convert any static values to variables as required. The hostname for example:

``` xml
<host-name>{{ hostname }}</host-name>
```

Partial variables to assist with streamlining a template, for example utilising a site-id number as part of the template:

``` xml
      <host-name>site-{{ site_id }}-fw</host-name>
```

and

``` xml
<interface>
  <name>irb</name>
  <unit>
    <name>0</name>
    <family>
      <inet>
        <address>
          <name>10.{{ site_id )).0.1/24</name>
        </address>
      </inet>
    </family>
  </unit>
</interface>
```

### Template Testing

Add the template to Sky Enterprise, under the configuration tab, and assign it to a new device (this can be a dummy device or a lab device initially). Provide the variable values when you do so, dummy data can be used for this.

From the Configuration Tab the new device will be listed as a ZTP Device, use the drop down action button to view the target XML Configuration.

Copy this XML configuration (i.e. the configuration merged from the template) from Sky Enterprise and trim the top and bottom leaving:

``` xml
    <configuration> 
            ... 
    </configuration>
```

Copy the XML config above onto a test SRX device. Its possible to use 'start shell command "vi /var/tmp/config.xml"' or copy the file using scp/sftp.

With `config.xml` in `/var/tmp/`:

```
    root@srx300# load factory-default
    warning: activating factory configuration
    root@srx300# set system root-authentication plain-text-password
    New password:
    Retype new password:
    [edit]
    root@srx300# commit and-quit
    root@srx300> op url /etc/phone-home/phcd_commit.slax filename /var/tmp/config.xml action merge format xml
    root@srx300> ...ename /var/tmp/config.xml action merge format xml
    [edit security zones security-zone customer]
    'interfaces ge-0/0/0.0'
        Interface ge-0/0/0.0 already assigned to another zone
    error: configuration check-out failed
```

Fix any commit issues in the config.xml file then re-test:

```
    root@srx300> ...x filename /var/tmp/config.xml action merge format xml
    commit complete
```

This mimics what ztp/phone-home does, as seen in these sample ZTP logs from an SRX:

```
     Feb  2 11:45:40 C(SM-phcd_platform_junos_commit): CMD /usr/sbin/cli -c "op url /etc/phone-home/phcd_commit.slax filename /var/tmp/phone-home/phcd_downloaded_cfg.xml action merge format xml"
```

### Software Upgrades via ZTP

It is possible to upgrade the Junos version during the ZTP process. This is done by inserting the following XML into the header of the XML template. This requires the Junos image to be hosted on a server accessible to both the device and the Sky Enterprise ZTP Servers (for secure URL proxying)

``` xml
    <device xmlns="http://juniper.net/zerotouch-bootstrap-server"> <unique-id>{{ serial }}</unique-id>
          <boot-image>
              <name>junos-srxsme-15.1X49-D150.2-domestic.tgz</name>
              <md5>84e810e4f68ca399d4b50d537ac0b413</md5>
              <sha1>0c41dfea61935920f82a698761377fc6186d6902</sha1>
              <path>http:||server|junos-srxsme-15.1X49-D150.2-domestic.tgz</path>
              <signature/>
          </boot-image>
          <configuration>
              <config>
              </config>
          </configuration>
    </device>
```

This could be extended to use variables too:

``` xml
<device xmlns="http://juniper.net/zerotouch-bootstrap-server"> <unique-id>{{ serial }}</unique-id>
<boot-image>
<name>{{ sw_image }}</name>
<md5>{{ sw_image_md5 }}</md5>
<sha1>{{ sw_image_sha1 }}</sha1>
<path>http:||{{ sw_server }}|{{ sw_image }}</path>
        <signature/>
          </boot-image>
          <configuration>
              <config>
                ...      
              </config>
          </configuration>
 </device>
```

### ZTP Dry Run

With a working template and resulting configuration attempt the ZTP process end-to-end. In testing, zeroise the SRX, confirm it has internet access after it has rebooted and configure traceoptions to observe phone-home behaviour (this will require a root password to be configured):

```
      set system root-authentication ...
      set system phone-home traceoptions flag all
      set system phone-home traceoptions file phone-home
```

commit this and use `monitor start phone-home` or `show log phone-home` to view the activity. Once authorised in Sky Enterprise all the phcd activity is logged to phone-home. XML or Junos syntax issues will be flagged in the logs.

### ZTP 'Hands Free'

With a fully tested template, a freshly unboxed SRX (or zeroised) should run through this process without any console intervention, and only require ZTP authorisation via Sky Enterprise to get it online and managed.

### EX Switches and NFX Devices

These devices follow the same principles.
