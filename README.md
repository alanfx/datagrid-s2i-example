# Data Grid Customized Openshift S2I Template

The ideia behind [this template](openshift/resources/datagrid71-postgresql-persistent-s2i.json) is demonstrate how you can customize the configuration of your Data Grid Cluster using the [Official Red Hat xPaaS Image for JBoss Data Grid 7.x](https://access.redhat.com/documentation/en-us/red_hat_jboss_data_grid/7.1/html-single/data_grid_for_openshift/).

This is just a an extension of the original Openshift Template for Data Grid 7.1 ([**`datagrid71-postgresql-persistent.json`**](https://github.com/jboss-openshift/application-templates/blob/master/datagrid/datagrid71-postgresql-persistent.json)). The original Data Grid xPaaS template offers a bunch of ([*parameters* and *environment variables*](https://access.redhat.com/documentation/en-us/red_hat_jboss_data_grid/7.1/html-single/data_grid_for_openshift/#jdg-configuration-environment-variables) you can use to cusomize the Data Grid during the Deployment process on Openshift Platform. **But at the moment it does not cover every configuration details you can use to setup your Data Grid Cluster. For exemple you can't configure/define things like: *cache base configurations*, *cache writting modes*, *memory based eviction policy*, change the *passivation mode* for a specifc jdbc-cache-store and so on...**

Holpefully the Data Grid xPaaS image is also a S2I Builder Image. So all the capabilities offered by the Openshift S2I approach can be leveraged!. See the [DATA GRID FOR OPENSHIFT](https://access.redhat.com/documentation/en-us/red_hat_jboss_data_grid/7.1/html-single/data_grid_for_openshift/#using_the_jdg_for_openshift_image_source_to_image_s2i_process) for more details.

As an extension [this template](openshift/resources/datagrid71-postgresql-persistent-s2i.json) combines the two capabilities to customize your Data Grid Deployment on Openshift:
 * the original xPaaS image template parameters; and
 * the s2i approach.

Basically you have to add/customize your Data Grid Config using the [`clustered-openshift.xml`](configuration/clustered-openshift.xml) file and than deploy it using the extended template.


See the config snippet I defined on the [`configuration/clustered-openshift.xml`](configuration/clustered-openshift.xml):
```xml
        <subsystem 
            xmlns="urn:infinispan:server:core:8.4" default-cache-container="clustered">
            <!-- placeholder changed to not be haldled by the image's launch scripts -->
            <!-- ##_IGNORE_INFINISPAN_CORE## -->

            <!-- @@ START S2I CUSTOM CONFIG @@ -->
            <!-- NOTE: DO NOT CHANGE THE NAME OF THE 'clustered' cache-container. 
                 It is referenced by the connectors' endpoint subsystem
                 auto generated by the original image lauch scripts...
                 
                 If you do so you have to remove the '' placholder from this file
                 and provide your own connectors' enpoint subsystem configuration!
                  -->
            <cache-container name="clustered" default-cache="demo">
                <transport channel="cluster" />
                <global-state/>

                <!-- cache configurations -->
                <distributed-cache-configuration name="base_dist_jdbc_cache_config" mode="SYNC">
                    <eviction strategy="LRU" size="10000"/>
                    <expiration lifespan="-1" max-idle="-1" />
                    <string-keyed-jdbc-store 
                        datasource="java:jboss/datasources/postgresql" 
                        passivation="false"
                        purge="false" 
                        preload="true" 
                        shared="false">
                        <string-keyed-table prefix="JDG">
                            <id-column name="id" type="VARCHAR"/>
                            <data-column name="datum" type="BYTEA"/>
                            <timestamp-column name="version" type="BIGINT"/>                            
                        </string-keyed-table>
                       <write-behind modification-queue-size="1024" thread-pool-size="1"/>
                    </string-keyed-jdbc-store>
                </distributed-cache-configuration>

                <distributed-cache name="demo" configuration="base_dist_jdbc_cache_config"/>
                <distributed-cache name="default_memcached" mode="SYNC"/>
            </cache-container>            
            <!-- @@ END   S2I CUSTOM CONFIG @@ -->

        </subsystem>
```

During the deployment the **s2i scripts** will copy your `configuration/clustered-openshift.xml` and than the launch scripts will process the original ``## PLACEHOLDERS ##`` according to the parameters you informed on template. 

> NOTE: if you do not want the lauch script process a specifc part of the `clustered-openshift.xml` file just modify the `## PLACEHOLDER ##` name. Like this `<!-- ##_IGNORE_INFINISPAN_CORE## -->`

See the steps below...

## Steps to deploy this sample Data Grid app

1. Create a new project/namespace on your Openshift Cluster (eg: local Minishift, CDK or `oc cluster up` if you are a Linux user)
```bash
oc login ...

oc new-project dg-s2i-demo
```

2. Import this template
```bash
oc create -f https://raw.githubusercontent.com/rafaeltuelho/datagrid-s2i-example/master/openshift/resources/datagrid71-postgresql-persistent-s2i.json
```

3. Create the `datagrid-app-secret` sample secret (with sample self-signed keystores) used for HTTPS and JGroups internal cluster secured communication.
```bash
oc create -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/secrets/datagrid-app-secret.json
```

4. Assign the `view` role to the `default` Service Account. This is requeired by the JGroups internal cluster communication.
```bash
oc policy add-role-to-user view system:serviceaccount:$(oc project -q):default -n $(oc project -q)
```

5. Create the app using Openshift web console (from catalog) or using `oc` CLI.

```bash
oc new-app --template='datagrid71-postgresql-persistent-s2i' \
--param=APPLICATION_NAME='mycustom-dg-cluster' \
--param=SOURCE_REPOSITORY_URL='https://github.com/rafaeltuelho/datagrid-s2i-example' \
--param=USERNAME='admin' \
--param=PASSWORD='Data@grid1' \
--param=JAVA_OPTS_APPEND='-Dcustom_opts_mark=custom-dg-cluster -Xms512m -Xmx512m'
```

> the template defines default values for the other supported parameters. If you need to change any of them see the template definition to get the name of the parameter.

### Testing your Data Grid using the REST Server

... from a command prompt:

* Create an entry into `DEMO` Cache
```bash
curl -v -k --basic --user 'admin':'Data@grid1' \
-X PUT \
https://secure-datagrid-app-demo-jdg.apps.[your local IP].nip.io/rest/DEMO/k1 \
-d "Hello World!"
```

* Get the entry value from the Cache
```bash
curl -v -k --basic --user 'admin':'Data@grid1' \
https://secure-datagrid-app-demo-jdg.apps.[your local IP].nip.io/rest/DEMO/k1
```

## Content

This sample repo contains the following files:

* `configuration`
  * [`clustered-openshift.xml`](configuration/clustered-openshift.xml): config file based on the original using the same placeholders (## PLACEHOLDER ##) processed by the image lauch script. **Add your customizations in this file**. 

  > The content of this directory will be copied by the Data Grid (xPaaS) s2i builder image, processed by the launch scripts and finally be placed into `/opt/datagrid/standalone/configuration` inside the container...

* `model`
  * [`original-placeholders-clustered-openshift.xml`](model/original-placeholders-clustered-openshift.xml): original `clustered-openshift.xml` config file processed by the image launch scripts during deployment. This is were the `## PLACEHOLDER ##` are replaced by the templates ENV VARs/parameters during deployment. **It's here just for reference...**
  * [`sample-datagrid71-template-generated-clustered-openshift.xml`](mode/sample-datagrid71-template-generated-clustered-openshift.xml): This is a sample config file after deployment (after being processed by the launch scripts). **It's here just for reference...**

* `openshift`
  * `resources`
    * [`datagrid71-postgresql-persistent-s2i.json`](openshift/resources/datagrid71-postgresql-persistent-s2i.json): **template that do the deployment magic of this sample!**