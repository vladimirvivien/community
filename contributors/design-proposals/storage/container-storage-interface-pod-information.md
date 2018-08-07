# Pod in CSI NodePublish request
Author: @jsafrane

## Goal
* Pass Pod information (pod name/namespace + service account) to CSI drivers in `NodePublish` request as volume attributes.
* Establish API where kubelet can keep track of which CSI drivers should receive Pod in NodePublish and which don't want it.

## Motivation
We'd like to move away from exec based Flex to gRPC based CSI volumes. In Flex, kubelet always passes `pod.namespace`, `pod.name`, `pod.uid` and `pod.spec.serviceAccountName` ("pod information") in every `mount` call. In Kubernetes community we've seen some Flex drivers that use pod or service account information to authorize or audit usage of a volume or generate content of the volume tailored to the pod (e.g. https://github.com/Azure/kubernetes-keyvault-flexvol).

CSI is agnostic to container orchestrators (such as Kubernetes, Mesos or CloudFoundry) and as such does not understand  concept of pods and service accounts. [Enhancement of CSI protocol](https://github.com/container-storage-interface/spec/pull/252) to pass "workload" (~pod) information from Kubernetes to CSI driver has met some of resistance. 

## High-level design
We decided to pass the pod information as `NodePublishVolumeRequest.volume_attributes`.

* Kubernetes passes pod information only to CSI drivers that explicitly require that information. These drivers are tightly coupled to Kubernetes and may not work or may require reconfiguration on other cloud orchestrators. It is expected (but not limited to) that these drivers will provider ephemeral volumes similar to Secrets or ConfigMap, extending Kubernetes secret or configuration sources.
* Kubernetes will not pass pod information to CSI drivers that don't know or don't care about pods and service accounts. It is expected (but not limited to) that these drivers will provide real persistent storage. Such CSI driver would reject a CSI call with pod information as invalid.

## Detailed design

### CSI enhancement
We don't need to change CSI protocol in any way. It allows kubelet to pass `pod.name`, `pod.uid` and `pod.spec.serviceAccountName` in [`NodePublish` call as `volume_attributes`]((https://github.com/container-storage-interface/spec/blob/master/spec.md#nodepublishvolume)). `NodePublish` is roughly equivalent to Flex `mount` call.

The only thing we need to do is to **define** names of the `volume_attributes` keys that CSI drivers can expect:
	*	`csi.kubernetes.io/pod.name`: name of the pod that wants the volume.
	*	`csi.kubernetes.io/pod.namespace`: namespace of the pod that wants the volume.
	*	`csi.kubernetes.io/pod.uid`: uid of the pod that wants the volume.
	*	`csi.kubernetes.io/serviceAccount.name`: name of the service account under which the pod operates. Namespace is the same as `pod.namespace`.

Note that these attribute names are very similar to [parameters we pass to flex volume plugin](https://github.com/kubernetes/kubernetes/blob/10688257e63e4d778c499ba30cddbc8c6219abe9/pkg/volume/flexvolume/driver-call.go#L55). Kubernetes overwrites these `volume_attributes` if they were already present in a CSI volume (either in a PV or in-line in a pod).

### Kubernetes 
Right now, we have a sidecar container called Driver Registrar that runs with a CSI driver container on every node (via DaemonSet). This Driver Registrar implements [plugin registration v1alpha1 API](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/pluginregistration/v1alpha1/api.proto)  and sends path to a CSI driver UNIX domain socket to kubelet. In kubelet, this socket is passed to [CSI driver plugin](https://github.com/kubernetes/kubernetes/blob/10688257e63e4d778c499ba30cddbc8c6219abe9/pkg/volume/csi/csi_plugin.go#L87), which then keeps track of registered CSI drivers and their sockets.

We need to extend this interface to pass also a flag if the CSI driver wants to receive pod information in `NodePublish` requests. Such flag can't be passed via CSI itself, because CSI itself does not know (and refuses to know) anything about pods or service accounts. Therefore we propose a new gRPC API between the CSI Driver Registar and kubelet.

Location: `pkg/volumes/apis/csiregistration/v1alpha1/api.proto`
```protobuf
package csiregistration;

import "github.com/gogo/protobuf/gogoproto/gogo.proto";

option (gogoproto.goproto_stringer_all) = false;
option (gogoproto.stringer_all) =  true;
option (gogoproto.goproto_getters_all) = true;
option (gogoproto.marshaler_all) = true;
option (gogoproto.sizer_all) = true;
option (gogoproto.unmarshaler_all) = true;
option (gogoproto.goproto_unrecognized_all) = false;

// CSIDriverInfo is the message sent from a CSI driver registrar to the Kubelet pluginwatcher for CSI driver registration
message CSIDriverInfo {
        // UNIX Socket with CSI driver endpoint
        string csi_endpoint = 1;
        // Configuration of kubelet behavior for this driver
        CSIDriverOptions options = 2;
}

message CSIDriverOptions {
        // Indicates if kubelet shall send Pod in NodePublish requests
        bool requires_publish_pod = 1;
}


// CSIDriverInfo is the empty request message from Kubelet
message CSIDriverInfoRequest {
}

// Registration is the service advertised by the Plugins.
service CSIRegistration {
        rpc GetCSIDriverInfo(CSIDriverInfoRequest) returns (CSIDriverInfo) {}
}
```

This API will be accompanied with `hack/update-generated-csi-registration.sh` and `hack/update-generated-csi-registration-dockerized.sh` to generate golang code for the API definition (based on `hack/update-generated-device-plugin*.sh`).


#### CSI volume plugin
* It already has a map of registered CSI drivers and their sockets. This map must be extended with a flag if a driver wants to receive pod information in `NodePublish`.
* It implements client side of `CSIRegistration` service. When the plugin gets a callback that a new CSI socket (created by Driver Registrar) was detected, it uses this socket to send `CSIDriverInfoRequest` to the Driver Registrar. The Driver Registrar either returns `CSIDriverInfo` with the flag or the Driver Registrar does not implement the method and we can assume that the corresponding CSI driver does not want to receive pod information.
* Based on the flag above, it passes pod information in `NodePublish` requests.

#### CSI Driver Registrar
It runs as a sidecar container in every pod with a CSI driver on every node. Each CSI driver runs its own CSI Driver Registrar.
* It has a new command line option `--requires-publish-pod` that admin (or CSI driver author) sets if the CSI driver needs pod information in `NodePublish`.
* It implements server part of `CSIRegistration` service, i.e. listens for  `GetCSIDriverInfo` call and returns `CSIDriverInfo` with the flag.


## End to end examples

### CSI driver wants pod information in `NodePublish`
1. CSI driver author provides documentation and templates that the driver need  pod information in `NodePublish`.
2. Admin installs CSI driver with CSI Driver Registrar `--requires-publish-pod`, e.g. by applying helm template from CSI driver author.
3. On a node, CSI Driver Registrar places UNIX domain socket to `/var/lib/kubelet/plugins/foo-reg.sock`.
4. Kubelet finds the socket there and calls `GetInfo` on it (no change from current behavior).
5. CSI Driver Registrar receives `GetInfo` call and passes CSI driver socket to kubelet (no change from current behavior).
6. Kubelet notifies CSI volume plugin that there is a new CSI driver installed and passes **both** CSI DriverRegistrar socket and CSI driver socket (no change from current behavior).
7. CSI volume plugin calls `GetCSIDriverInfo` on the CSI Driver Registrar socket.
8. CSI Driver Registrar, based on `--requires-publish-pod` presence on command line, returns `CSIDriverInfo` with `requires_publish_pod = true`.
9. CSI volume plugin stores this flag for the CSI driver.

When a volume is being mounted for a pod:

1. In CSI volume plugin `MountDevice()`, the plugin checks if the CSI driver supports `NodeStageVolume` and calls it without any pod information. `MountDevice()` and `NodeStageVolume` is called only once per volume if the volume is used by multiple pods. Passing just the first pod to it is useless.
2. In CSI volume plugin `SetUpAt()`, the plugin checks if the CSI driver requires pod information and passes it as `volume_attributes` defined above in CSI `NodePublishVolume` call.

### CSI driver uses old CSI Driver Registrar image
When a CSI driver uses CSI Driver Registrar that does not implement `CSIRegistration` service.
1. Admin installs CSI driver with old CSI Driver Registrar.
2. On a node, CSI Driver Registrar places UNIX domain socket to `/var/lib/kubelet/plugins/foo-reg.sock`.
3. Kubelet finds the socket there and calls `GetInfo` on it (no change from current behavior).
4. CSI Driver Registrar receives `GetInfo` call and passes CSI driver socket to kubelet (no change from current behavior).
5. Kubelet notifies CSI volume plugin that there is a new CSI driver installed and passes **both** CSI DriverRegistrar socket and CSI driver socket (no change from current behavior).
6. CSI volume plugin calls `GetCSIDriverInfo` on the CSI Driver Registrar socket.
7. CSI Driver Registrar does not implement `GetCSIDriverInfo` and returns an error.
8. CSI volume plugin marks the corresponding CSI driver as it does not need pod information.

## Alternatives
* Add a new flag to [`PluginInfo` message](https://github.com/kubernetes/kubernetes/blob/10688257e63e4d778c499ba30cddbc8c6219abe9/pkg/kubelet/apis/pluginregistration/v1alpha1/api.proto#L17). This way, we would not need to implement new gRPC service. On the other hand, `PluginInfo` is shared among all kubelet plugins (i.e. device plugins and CSI drivers) and such flag would pollute API for device plugins.
* Add flag to CSI itself. It was refused in CSI community, they don't want to pass flags specific to Kubernetes.

## Future enhancements
In future we may add new flags / options to `CSIDriverInfo` message as Kubernetes CSI volume plugin may require more configuration how to work with each registered CSI driver.

