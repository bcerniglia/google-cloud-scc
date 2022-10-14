# Google Security Command Center - Asset Inventory

## Prerequisites

Within a project you have gcloud configured as your default enable the following API   
**Security Command Center API** - securitycenter.googleapis.com

At the org level you will need this permission assigned to your username/svc_acct   
**Security Center Admin Viewer** - roles/securitycenter.adminViewer

The ability to run the google-cloud-sdk with the beta components installed

## Basic Command Structure

Asset listing is done either at the project, folder, or, organization level. The volume of the output increases <span style="color:red">**drastically**</span> as
you work your way up the hierarchy.

List all assets within a projects  
```gcloud beta scc assets list projects/prj-example-1234```

List all assets under a folder within our organization  
```gcloud beta scc assets list folders/123456789012```

List all assets under our organization   
```gcloud beta scc assets list 424183677400```

## Finding and Filtering Assets

List of supported [searchable assets](https://cloud.google.com/security-command-center/docs/supported-asset-types) within Security Command Center. 
Take note of the two separate services being referenced in that list, Cloud Asset Inventory and Security Command Center


**List out every resource within a project and flatten it - flattening will help understand the key/value nature of this output**  
```
gcloud beta scc assets list projects/prj-example-1234 --format=flattened
```


```json
asset.canonicalName:                                                 projects/2222111122222/assets/6584984351265465464
asset.createTime:                                                    2022-03-01T05:09:31.186Z
asset.iamPolicy:                                                     {}
asset.name:                                                          organizations/424183677400/assets/6584984351265465464
asset.resourceProperties.description:                                Default bucket
asset.resourceProperties.lifecycleState:                             ACTIVE
asset.resourceProperties.name:                                       projects/2222111122222/locations/global/buckets/_Default
asset.resourceProperties.retentionDays:                              30
asset.securityCenterProperties.folders[0].resourceFolder:            //cloudresourcemanager.googleapis.com/folders/3331111333333
asset.securityCenterProperties.folders[0].resourceFolderDisplayName: fldr-1234
asset.securityCenterProperties.folders[1].resourceFolder:            //cloudresourcemanager.googleapis.com/folders/123456789012
asset.securityCenterProperties.folders[1].resourceFolderDisplayName: devops
asset.securityCenterProperties.resourceDisplayName:                  projects/2222111122222/locations/global/buckets/_Default
asset.securityCenterProperties.resourceName:                         //logging.googleapis.com/projects/2222111122222/locations/global/buckets/_Default
asset.securityCenterProperties.resourceOwners[0]:                    user:bcerniglia@owletcare.com
asset.securityCenterProperties.resourceParent:                       //cloudresourcemanager.googleapis.com/projects/2222111122222
asset.securityCenterProperties.resourceParentDisplayName:            prj-example-1234
asset.securityCenterProperties.resourceProject:                      //cloudresourcemanager.googleapis.com/projects/2222111122222
asset.securityCenterProperties.resourceProjectDisplayName:           prj-example-1234
asset.securityCenterProperties.resourceType:                         google.logging.LogBucket
asset.securityMarks.name:                                            organizations/424183677400/assets/6584984351265465464/securityMarks
asset.updateTime:                                                    2022-03-01T05:09:31.186Z
```

**List every resource within a project that has a resourceType: google.logging.LogBucket**

```
gcloud beta scc assets list projects/prj-example-1234 \
--filter="security_center_properties.resource_type = \"google.logging.LogBucket\"" --format=flattened
```

```json
asset.canonicalName:                                                 projects/2222111122222/assets/15883817191309541680
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~OMITTED
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
asset.securityCenterProperties.resourceProject:                      //cloudresourcemanager.googleapis.com/projects/2222111122222
asset.securityCenterProperties.resourceProjectDisplayName:           prj-example-1234
asset.securityCenterProperties.resourceType:                         google.logging.LogBucket
asset.securityMarks.name:                                            organizations/424183677400/assets/15883817191309541680/securityMarks
asset.updateTime:                                                    2022-03-01T05:09:31.186Z
---
asset.canonicalName:                                                 projects/2222111122222/assets/6584984351265465464
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~OMITTED
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
asset.securityCenterProperties.resourceProject:                      //cloudresourcemanager.googleapis.com/projects/2222111122222
asset.securityCenterProperties.resourceProjectDisplayName:           prj-example-1234
asset.securityCenterProperties.resourceType:                         google.logging.LogBucket
asset.securityMarks.name:                                            organizations/424183677400/assets/6584984351265465464/securityMarks
asset.updateTime:                                                    2022-03-01T05:09:31.186Z
```

## Finding and Applying Storage Lifecycles
This will search for all buckets under a folder with the folder number 123456789012

```
gcloud beta scc assets list folders/123456789012 \
--filter="security_center_properties.resource_type = \"google.cloud.storage.Bucket\"" \
--format="table(asset.securityCenterProperties.folders[0].resourceFolderDisplayName, \
asset.securityCenterProperties.resourceProjectDisplayName,\
asset.securityCenterProperties.resourceType,\
asset.resourceProperties.id,\
asset.resourceProperties.storageClass,\
asset.resourceProperties.lifecycle)" --sort-by=asset.securityCenterProperties.resourceProjectDisplayName
```

```
RESOURCE_FOLDER_DISPLAY_NAME  RESOURCE_PROJECT_DISPLAY_NAME  RESOURCE_TYPE                ID                        STORAGE_CLASS  LIFECYCLE
fldr-1234           prj-example-1234        google.cloud.storage.Bucket  bkt-state-1234  STANDARD       {"rule":[]}
fldr-5678           prj-example-5678        google.cloud.storage.Bucket  bkt-state-5678  STANDARD       {"rule":[]}
```

Add a 30day archive lifecycle on bucket 'bkt-state-1234' which currently resides in project prj-example-1234   
*A set of example lifecycle rules are kept in json format in the 'lifecycles' directory*  
```
gsutil lifecycle set lifecycles/archive_30d.json gs://bkt-state-1234
```

```
Setting lifecycle configuration on gs://bkt-state-1234/...
```

Rerun our asset search from the first step and we should see the following:
```
RESOURCE_FOLDER_DISPLAY_NAME  RESOURCE_PROJECT_DISPLAY_NAME  RESOURCE_TYPE                ID                        STORAGE_CLASS  LIFECYCLE
fldr-1234           prj-example-1234        google.cloud.storage.Bucket  bkt-state-1234  STANDARD       {"rule":[{"action":{"storageClass":"ARCHIVE","type":"SetStorageClass"},"condition":{"age":30.0,"matchesPrefix":[],"matchesStorageClass":[],"matchesSuffix":[]}}]}
fldr-5678           prj-example-5678        google.cloud.storage.Bucket  bkt-state-5678  STANDARD       {"rule":[]}
```

Remove any lifecycle rule applied to a bucket

```
gsutil lifecycle set lifecycles/empty.json gs://bkt-state-1234
```

Rerun our asset search from the first step and we should see the following:
```
RESOURCE_FOLDER_DISPLAY_NAME  RESOURCE_PROJECT_DISPLAY_NAME  RESOURCE_TYPE                ID                        STORAGE_CLASS  LIFECYCLE
fldr-1234           prj-example-1234        google.cloud.storage.Bucket  bkt-state-1234  STANDARD       {"rule":[]}
fldr-5678           prj-example-5678        google.cloud.storage.Bucket  bkt-state-5678  STANDARD       {"rule":[]}
```

