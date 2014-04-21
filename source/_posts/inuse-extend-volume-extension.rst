g`" This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================================
Enable extend of in-use/attached volumes for libvirt/qemu driver
=================================================================

https://blueprints.launchpad.net/nova/+spec/inuse-extend-volume-extension

There will be no need for downtime of the instance workflow if we enable 
extend of in-use/attached volumes. And a potential benefit for instances 
that are enabled for disaster replication is that there will be no need to
resync the instance after modifying the disk, and no need to delete and 
re-enable replication.

There is a blueprint for the Cinder part of this at [1]


Problem description
===================

In order to support extend of in-use volumes for libvirt/qemu, it requires a 
number of things to happened and some of which need Nova's assist:

* Cinder: an in-use/attached volume can be extended or not depends on the 
  capability and storage allocation rules of Cinder backend. 

* Cinder & Nova: for some types of volume, such as non-file based volumes, 
  Cinder first extends the volume, then Nova do the rescan action to make the
  volume's new size to be refreshed on compute node which running the instance.

* Nova: make the volume's update passed into the guest instance. Libvirt/Qemu 
  driver has provided a "block_resize" interface to support this scenario.


Proposed change
===============

Add extension to Nova to support actions as follows:

* Support the rescan API. Rescan will make the volume's new size to be 
  gotten on compute node. Rescan will be called by Cinder after extending
  the volume.

* Support the get_blockdev API. Get_blockdev will return the volume's size info
  on compute node which running the instance. Get_blockdev will be called by 
  Cinder circularly to judge whether the rescan action has already done. 
 
* Support the assist_extend API. Assist_extend will finally call libvirt's 
  block_resize interface to make the volume's update passed into the instance.
  Assist_extend will be called by Cinder after pull the result of get_blockdev.


Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

Add "os-online_extend_volume" extension which has APIs as follows.

1. Rescan volume API

* rescan

  * Rescan volume to get the updated info

  * Method type: PUT

  * Normal http response code(s): 202

  * Expected error http response code(s)

    * 404: Instance specified is not found

  * URL: v2/{tenant_id}/servers/{server_id}/os-online_extend_volume

  * Parameters which can be passed via the url: None

  * JSON schema definition for the body data if allowed::

    {
        "onlineExtendVolume": {
            "volume_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        }
    }

  * JSON schema definition for the response data if any: None

* Example use case::

  Request: PUT v2/f66d7df4b59f4ab98a358a0dab0b238f/servers/14540a2f-204b-4a30\
          -a61b-e069bf243c87/os-online_extend_volume
            {
                "onlineExtendVolume": {
                    "volume_id": "076cc7cd-1239-4e9f-a1d3-06cb50b107a4",
                }
            }

  Response: HTTP 1.1 202/Accepted

* Policy: this API need admin role

#. Get Blockdev API

* get_blockdev

  * Return the updated size info of volume

  * Method type: GET

  * Normal http response code(s): 200

  * Expected error http response code(s)

    * 404: Instance specified is not found

  * URL: v2/{tenant_id}/servers/{server_id}/os-online_extend_volume/{volume_id}

  * Parameters which can be passed via the url: None

  * JSON schema definition for the body data if allowed: None

  * JSON schema definition for the response data if any::

    {
        "volume_blockdev":{
            "id": id,
            "bytes": blockdev_bytes
        }
    }

* Example use case::

  Request: GET v2/f66d7df4b59f4ab98a358a0dab0b238f/servers/14540a2f-204b-4a30\
          -a61b-e069bf243c87/os-online_extend_volume/076cc7cd-1239-4e9f-a1d3-\
          06cb50b107a4

  Response: HTTP 1.1 200/OK
        {
            "volume_blockdev":{
                "id": 076cc7cd-1239-4e9f-a1d3-06cb50b107a4,
                "bytes": 4294967296
            }
        }

* Policy: this API need admin role

#. Assist extend API

* assist_extend

  * Extend volume through assist of hypervisor driver

  * Method type: POST

  * Normal http response code(s): 200

  * Expected error http response code(s)

    * 404: Instance specified is not found

    * 409: Invalid state of instance

    * 501: Not supported by hypervisor driver

    * 500: Extend failed by hypervisor driver for other reasons, including
      read-only volume, drain IO failed, etc.

  * URL: v2/{tenant_id}/servers/{server_id}/os-online_extend_volume/{volume_id}

  * Parameters which can be passed via the url: None

  * JSON schema definition for the body data if allowed::

    {
        "onlineExtendVolume": {
            "bytes": xxx
        }
    }

  * JSON schema definition for the response data if any: None

* Example use case::

  Request: POST v2/f66d7df4b59f4ab98a358a0dab0b238f/servers/14540a2f-204b-4a30\
          -a61b-e069bf243c87/os-online_extend_volume/076cc7cd-1239-4e9f-a1d3-\
          06cb50b107a4
            {
                "onlineExtendVolume": {
                    "bytes": 4294967296
                }
            }

  Response: HTTP 1.1 200/OK

* Policy: this API need admin role


Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <zhangleiqiang@huawei.com>


Work Items
----------

* Add rescan, get_blockdev and assist_extend interface to virt driver. Add 
  corresponding methods for these interfaces to compute.api, compute.manager 
  and compute.rpcapi
  
* Add os-online_extend_volume extension 
  
* Add rescan implementation to libvirt driver 

* Add get_blockdev implementation to libvirt driver
 
* Add assist_extend implementation to libvirt driver


Dependencies
============

* There is a blueprint for the Cinder part of this at [1].

* Libvirt assisted extend feature requires libvirt version >= 0.9.8 and 
  qemu >= 1.5


Testing
=======

Unit tests are sufficient because of all methods are new added to Nova without
changing exists methods.


Documentation Impact
====================

None. 

The impact exists on Cinder docs.


References
==========

* [1]https://blueprints.launchpad.net/cinder/+spec/inuse-extend-volume-extension
