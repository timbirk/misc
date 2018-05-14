# misc
## Contents
- [Artifactory](#artifactory)
- [Mcollective](#mcollective)
- [Jenkins](#jenkins)
- [Puppet](#puppet)
- [Terraform](#terraform)
- [Bash](#bash)

***
## Artifactory
### YUM Metadata Recalculation
Q: Artifactory returns `Unable to perform immediate YUM metadata calculation on a repository with auto-async calculation enabled.`

A: Artifactory are often quite slow to regenerate metadata when RPMs are uploaded. To mitigate this we make an explicit API call to trigger an update, however this requires that auto metadata generation be disabled in the repository settings. If you see this error ensure `Auto Calculate RPM Metadata` is **unticked** in the repo config.

### YUM Metadata Corruption
Q: Getting 404 from artifactory while trying to checkout repos.
```eg ..primary.xml.gz: [Errno 14] HTTPS Error 404 - Not Found```

A: Looks like there are some server side issues during the RPM metadata calculation. The old metadata are not replaced by the new ones. Issue is raised with artifactory but a solution is to manually delete the *repodata* folder from the artifactory UI and recalculate the metadata. Recalculating the metadata will create a new repodata folder. After that a ```yum clean all && yum repolist``` should work as expected.

To automate this process in Jenkins jobs add something like this before triggering a metadata recalculation:
```
cat /etc/jenkins/credentials.prop | grep 'password=' | sed 's/password=//' | sed "s/^/user: ${ARTIFACTORY_USERNAME}:/" | \\
                curl -K - -w "%{http_code}" -X DELETE https://repos.jfrog.io/repos/${DST_REPO}/repodata | grep 204
```
This ensures that the repodata is re-generated from scratch.

## Mcollective
### Unknown Validator
Q: mcollective returns `Cannot validate input package: Unknown validator: 'XXXX'` for a node.

A: Restart mcollective on the node

### Output time since last Puppet run nicely
Q: The JSON from the last_run_summary action is rather verbose. Can I filter it to see how long it's been since the last Puppet run on each node?

A: YES! Use a few commands for this:
```
mco rpc --agent puppet --action last_run_summary --argument parse_log=true --json | \
  jq -r '.[] | .sender + " " + (.data.since_lastrun | tostring)' | \
  sort -r -k2,2 | column -t

Output:
repocache-i0e0cc094111d39993          534
sensu-eu-west-1a                      519
grafana-i02be8b239e34a95a3            517
bastion-i095791cd5b6d2993f            515
sensu-eu-west-1b                      503
logstash_esmaster-i0905eeb26486e69be  502
kibana-i016ca9ed95e4c43cd             500
influx-i0adb4f138f600070d             496
uchiwa-i0772c28bfc8cb564b             491
sensu-eu-west-1c                      489
mcorabbit-i0cfec6d08546a7605          470
bastion-i010c10c167d5806a1            466
bastion-i0b2b62c6c2fd27775            460
dashing-i0e6591bb12690f715            457
``` 
Now you have a nicely formatted table in descending order.
 
## Jenkins
### Seed Could Not Find ?.jar
Q: Help! I'm getting the following error on my `${::product}-seed` job:
```
FAILURE: Build failed with an exception.

* What went wrong:
Could not resolve all dependencies for configuration ':testCompileClasspath'.
> Could not find bootstrap-core-assets.jar (org.jenkins-ci.ui:bootstrap:1.3.2).
   Searched in the following locations:
      https://jcenter.bintray.com/org/jenkins-ci/ui/bootstrap/1.3.2/bootstrap-1.3.2-core-assets.jar
```

A: Maven is trying to resolve dependencies which no longer exist in various repos. Refactor your repositories in `${::product}-infra/jenkins/gradle.build` to something like this:

```
repositories {
    maven { url 'http://nexus.xwiki.org/nexus/content/groups/public' }
    mavenCentral()
    maven { url 'https://repo.jenkins-ci.org/releases/' }
    jcenter()
}
```
### List Plugins for Hiera
Q: HELP!! My Jenkins plugins get updated / have been manually installed... I'd like to pin them in Jenkins and update them in a controlled way, is there some way to get the versions of packages and dependencies that is better than the plugin manager UI?

A: YES! Using the Groovy script interface at: http://url_to_jenkins/script run this:
```
println "profile_jenkins::plugins:"
jenkins.model.Jenkins.instance.pluginManager.plugins.findAll({ !(it.shortName ==~ /swarm/) }).sort().each {
  println "  ${it.shortName}:\n    version: '${it.version}'"
}
```
You can use that output in your hiera to pin plugins to the current good plugin versions, dependencies and any manually installed plugins (it happens 😢 ). Note: the swarm plugin is installed via `profile_jenkins` you can set it's version in hiera with: `profile_jenkins::slave_version: '<<version>>'`.

### List Plugins for DSL / Seed Job
Q. HELP! I added DSL for specific plugins and now my seed jobs / tests are broken due to missing plugins. I know I need to add them as `testPlugins` but there's a big chain of dependencies and it's taking too long to manually type. Can I get a list of the plugins I have installed to add as `testPlugins`?

A: Sure you can! As above the script interface is also useful for that. Runt he following:
```
jenkins.model.Jenkins.instance.pluginManager.plugins.findAll({ !(it.shortName ==~ /job-dsl|structs|build-monitor-plugin/) }).sort().each {
  println "testPlugins '${it.manifest.mainAttributes.getValue("Group-Id")}:${it.shortName}:${it.version}'"
}
```
Now you can replace the `testPlugins` list you have with all the plugins your CI install currently has installed. Note: Currently (Nov 2017) build-monitor-plugin has a problem in it's POM so we filter that out.

### Generate Encrypted Credentials for credentials.xml
Q. I know that some plugins simply base64 encode credentials and store them in their configuration xml but Jenkins seems to actually encrypt the values in credentials.xml. Before storing them in hiera / eyaml to use in a template I need to encrypt them **OR** I need to decrypt them as we've lost the original values / they've been manually created and forgotten. Is there any way to encrypt / decrypt values on my Jenkins instance?

A. YES! Yes there is, and you guessed it... The script interface can help you here. Here's an example of encryption and decryption:
```groovy
// Encrypt credentials

myPassword = "Hello Jenkins!"

encryptedPassword = hudson.util.Secret.fromString(myPassword).getEncryptedValue()

println(encryptedPassword)

// decrypting a password
println(hudson.util.Secret.decrypt(encryptedPassword))
```

### Karma Testing / Chrome / Docker Selenium
Q(1). Our frontend devs want to use Karma / chrome to run test in the CI pipeline. The developer tried to get it working but they get errors like:

```
[Unit Tests] [INFO] [31m02 01 2018 10:12:07.799:ERROR [launcher]: [39mChrome stderr: /home/jenkins-slave/workspace/build-pull-requests_TP6-491-AK4NY6D42CGVT6HIPBWT5U6IIONYUVAD4SALD4DDSXAY6CWBV6MA/talpay-user-static/node_modules/puppeteer/.local-chromium/linux-515411/chrome-linux/chrome: error while loading shared libraries: libX11.so.6: cannot open shared object file: No such file or directory
```

A(1). You'll notice that karma is trying to run a local binary pulled in by npm / node. This works fine on a dev's laptop and on your macbook but it won't work on our Linux servers as we don't install X11 (which is usually used for GUI applications). The easiest way to proceed is to either run a `selenium/standalone-chrome:x.x.x` docker container on your CI slave(s) or if you need something heavier spin up a Dockerized selenium hub cluster on dedicated instances and configure karma to use the selenium / webdriver hub. It's reasonably simple to do and well documented on the internet.

Q(2). I paired with the developer and we got everything in place but the tests still don't run, it sits doing nothing and then there's a new error:

```
02 01 2018 17:36:12.952:WARN [launcher]: Chrome via Remote WebDriver have not captured in 60000 ms, killing.
02 01 2018 17:36:13.022:INFO [WebDriver]: Killed Karma test.
02 01 2018 17:36:13.028:INFO [launcher]: Trying to start Chrome via Remote WebDriver again (1/2).
02 01 2018 17:37:13.029:WARN [launcher]: Chrome via Remote WebDriver have not captured in 60000 ms, killing.
02 01 2018 17:37:13.109:INFO [WebDriver]: Killed Karma test.
02 01 2018 17:37:13.113:INFO [launcher]: Trying to start Chrome via Remote WebDriver again (2/2).
02 01 2018 17:38:13.118:WARN [launcher]: Chrome via Remote WebDriver have not captured in 60000 ms, killing.
02 01 2018 17:38:13.185:INFO [WebDriver]: Killed Karma test.
02 01 2018 17:38:13.188:ERROR [launcher]: Chrome via Remote WebDriver failed 2 times (timeout). Giving up.
```

A(2). Mysterious... This is because karma listens on a port locally (9876 by default) and it expects a browser (webdriver in this case) to connect to a uri (`17:37:13.852 INFO - To upstream: {"url":"http://localhost:9876/?id=55234275"}`) that it passes to the browser.

Now that selenium-chrome is running inside a docker container, it tries to connect to port 9876 on the container. We need it to connect to the host that karma is running on. 

We can override the hostname for the uri passed into the browser (Selenium / WebDriver) with `config.hostname` and with nodejs it's quite easy to get the local IP address of the host / machine that karma is running on. If your dev's have followed standard practices you probably have a karma.conf.ci.js looking something like this:

```js
module.exports = function (config) {

    var webdriverConfig = {
        hostname: 'localhost',
        port: 4444
    }

    config.singleRun = true;
    config.customLaunchers = {
       'ChromeWebdriver': {
           base: 'WebDriver',
           config: webdriverConfig,
           browserName: 'Chrome',
           name: 'Karma',
           pseudoActivityInterval: 30000
       }
    };
    require('./src/karma.conf.js')(config, ['ChromeWebdriver']);
};
```

We can add a few lines to get the local IPV4 address and pass it in to the selenium browser:

```js
module.exports = function (config) {

    // Get local IP for selenium runner
    var address,
    ifaces = require('os').networkInterfaces();
    for (var dev in ifaces) {
       ifaces[dev].filter((details) => details.family === 'IPv4' && details.internal === false ? address = details.address: undefined);
    }

    var webdriverConfig = {
        hostname: 'localhost',
        port: 4444
    }

    // set hostname part of url to host IP address
    config.hostname = address;
    config.singleRun = true;
    config.customLaunchers = {
       'ChromeWebdriver': {
           base: 'WebDriver',
           config: webdriverConfig,
           browserName: 'Chrome',
           name: 'Karma',
           pseudoActivityInterval: 30000
       }
    };
    require('./src/karma.conf.js')(config, ['ChromeWebdriver']);
};
```

Now karma will pass in an upstream like: `12:29:26.001 INFO - To upstream: {"url":"http://10.228.222.100:9876/?id=72766741"}` which the selenium-chrome container can connect to. Happy testing!

## Puppet

### Custom facts with hiera lookup fallback

Q. We override a configuration value by setting (PUT) and unsetting (DELETE) a value in Consul KV, we want to use this value as a custom fact but fallback to a default value that will be different depending on {role, environment, ecosystem}. How can we do this? 

A. It's trivial to lookup the value from consul with a simple bash custom fact using curl. However, to set a default value from hiera if consul or the value isn't there you will need to do some funky escaping:

```bash
#!/usr/bin/env bash

#
# Set a configuration value in Consul, fallback to a hiera lookup 
#

echo "custom_fact_1=$(curl -f -s http://localhost:8500/v1/kv/custom-value-1?raw || echo '"%{hiera('\''hiera_value_1'\'')}"')"
echo "custom_fact_2=$(curl -f -s http://localhost:8500/v1/kv/custom-value-2?raw || echo '"%{hiera('\''hiera_value_2'\'')}"')"
```
Now when the consul value doesn't exist you get a 404 and your script will return something like: `custom_fact_x="%{hiera('hiera_value_x')}"` which hiera obeys and returns the value of the hiera key. 

Things worth noting: `-f` in the curl command is needed to return non-zero exit codes for non 200 HTTP codes. Escaping quotes in echo's in command substitution in bash is painful.

## Bash

### Generate a random hiera encrypted password. 
Q: I want a quick way to generate a random, safe password and hiera encrypt it. How can I do it?

A: Use this one-liner in your Puppet repo:
```bash
eyaml encrypt --string=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 16 | head -n 1)
```
You don't even get to see what the password is. 

### Get a map of AWS instance_type => on-demand price suitable for terraform variables.
Q. I can see a great list of on demand prices at: https://github.com/powdahound/ec2instances.info/tree/master/www (instances.json). It'd be great to use the on-demand prices as a map so I can look up the prices for things like spot_prices in Terraform. How?

A. There's a tool called `jq` that is useful for parsing, filtering and manipulating json. The script below is one approach to getting the map:
```sh 
#!/usr/bin/env bash

# spot_prices.sh - generate a default map suitable for Terraform vars

set -e -o pipefail

# Set this to whatever you're happy to pay as max spot_price.
# Example: 1.5 if you're happy paying up to 1.5x on demand price which can help with small price spikes.
# Note that the default spot_pric limit AWS sets is the same as the on demand price.
# If you specify above on demand AWS will silently set the spot_price to the on-demand price. 
multiple=1
aws_region="eu-west-1"

json_data=$(curl -s --fail https://raw.githubusercontent.com/powdahound/ec2instances.info/master/www/instances.json)

# Build the map
echo "spot_prices = {"
echo $json_data | jq -r '.[] |
	.instance_type as $it |
	.pricing | to_entries[] |
	select(.key == "'${aws_region}'") |
		"  \"" + $it + "\" = \"" +
        (.value.linux.ondemand | tonumber * '${multiple}' | tostring | .[0:5]) + "\""' | sort
echo "}"
```
and the output (truncated):
```
spot_prices = {
  "c1.medium" = "0.148"
  ....
  ....
  ....
  "x1e.4xlarge" = "4"
  "x1e.8xlarge" = "8"
  "x1e.xlarge" = "1"
}
```
