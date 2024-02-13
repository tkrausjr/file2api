# File2api
This repo simulates a simple application comprised of three pieces
1. File Server that serves a CSV file for download (This simulates an artifact server or git repo) but we are using nginx and http.
2. Kubernetes Job that downloads the CSV file and stores it on a Persistent Volume for later use in the K8s cluster.
3. A simulated (API Server) that will access the CSV file on the same PV and expose it from a webserver endpoint as a scalable K8s Deployment.

## 0 - Checkout this git repo

## 1 - Deploy the Simulated FileServer holding file 
- (Optional) Build the docker image for the webserver that will serve the CSV file. Upload the image to your local registry.
	- If you can download or access Docker Hub you can skip to the webserver deployment below. 
```shell
cd ./file2api/01-file-fs
docker build -t file-nginx-fs .
docker tag file-nginx-fs mytkrausjr/file-nginx-fs:v3
docker push mytkrausjr/file-nginx-fs:v3
```

- Deploy the webserver that houses the CSV file to Kubernetes as a K8s Deployment and Service to simulate the CSV Webserver.

```shell
kubectl apply -f . 
kubectl get po,svc
```

TEST:
- From Outside the Cluster, from a browser with access to the AVI VIP Network
	- Get the EXTERNAL-IP from the Service Above 	
	- http://<EXTERNAL-IP>:8888/data-file.csv
	- Success is the data-file.csv file is downloaded locally.
- From Inside the K8s Cluster
```shell
kubectl run busybox-wget-2 --rm=true --image=harbor-repo.vmware.com/dockerhub-proxy-cache/library/busybox --restart=Never \
-it -- wget -O- http://file-fs-svc.default.svc.cluster.local:8888/data-file.csv
```
## 2 - Deploy the Kubernetes Job and PVC that will download fhe file and store on a persistent volume.
```shell
cd ../02-file-dwnld
kubectl apply -f data-file-pvc.yaml
kubectl apply -f file-download-job.yaml
kubectl get job
	NAME            COMPLETIONS   DURATION   AGE
	file-download   1/1           14s        19h
```

## 3 - Deploy the Kubernetes Pod that will mount the PV load the file and expose it via website / Spring webserver
- (Optional) Build the docker image for the API Server that will serve the data in the CSV file. Upload the image to your local registry.
	- If you can download or access Docker Hub you can skip to the API Server K8s deployment below. 
```shell
cd ./file2api/03-file-api
docker build -t file-nginx-api:v7 .
docker tag file-nginx-api:v7 mytkrausjr/file-nginx-api:v7
docker push mytkrausjr/file-nginx-api:v7
```
- Deploy the API Server that will mount the CSV file downloaded to the Persistent Volume in Kubernetes as a K8s Deployment and Service to server the data in the CSV via API.
```shell
cd ../03-file-api
kubectl apply  -f .
	deployment.apps/file-api created
	service/file-api-svc created
kubectl get po,svc -l app=file-api
	NAME                            READY   STATUS    RESTARTS   AGE
	pod/file-api-755d78db88-cmpsl   1/1     Running   0          37m
	NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)          AGE
	service/file-api-svc   LoadBalancer   198.57.154.69   192.168.201.116   8888:32499/TCP   37m
```

- TEST
	- With a Browser - Navigate to ````http://<EXTERNAL-IP>:8888/data-file.csv````
 		- This will show you the CSV data (Tested in Chrome) 	
 	- With CURL in a Terminal
```shell     
curl http://192.168.201.119:8888/data-file.csv | column -t -s, | less -S

source-code  dest1-name    dest1-code  dest1-zip  dest2-name    dest2-code  dest2-zip  air  ground
AAR          Fort Solonga  FRG         11875      Jostons       GHT         12356      no   yes
ABD          Boston        BJS         21675      Barryville    YTY         19876      yes  yes
WEN          Reneau        REG         11768      Jensen        JHY         15609      no   yes
DRU          Johnsonville  JHY         16728      Ponston       UUU         12398      yes  yes
FAD          Unitas        UUU         11875      Ion           JKK         19234      yes  yes
BKD          Jenkins       JKK         11875      Yankee        BVC         11875      no   yes
ORG          Berryville    BVC         11785      Klingon       AAR         21675      no   yes
FRE          Arkass        AAR         11456      Grayson       KLJ         11768      no   yes
BAK          Avin          ABD         23567      Yellow        GHT         16728      no   yes
FRG          Wenston       WEN         14356      Jennings      YTY         11875      yes  yes
BJS          Drucker       DRU         12356      Uncle         JHY         11875      yes  yes
REG          Frensh        FAD         19876      Jostons       UUU         11785      yes  yes
JHY          Bayview       BKD         15609      Barryville    JKK         11456      no   yes
UUU          Eweon         EWQ         12398      Arkenstone    BVC         23567      no   yes
JKK          Askin         ASD         19234      Aberville     AAR         12398      yes  yes
BVC          Riding        FDS         11875      Berryville    ABD         19234      yes  yes
ASD          Shore         FGH         21675      Arkass        FRG         11875      no   yes
JYU          Dune          FGH         11768      Avin          BJS         21675      no   yes
QWE          Jensen        JHGF        16728      Wenston       REG         11768      no   yes
EWQ          Ponston       POI         11875      Drucker       JHY         16728      no   yes
ASD          Ion           IOP         11875      Frensh        UUU         11875      yes  yes
FDS          Yankee        YJU         11785      Bayview       JKK         11875      yes  yes
FGH          Klingon       KLJ         11456      Eweon         BVC         11785      yes  yes
JHGF         Grayson       GHT         23567      Askin         AAR         11456      no   yes
POI          Yellow        YTY         14356      Riding        ABD         23567      no   yes
IOP          Jennings      JHY         12356      Shore         WEN         14356      yes  yes
YJU          Uncle         UUU         19876      Dune          DRU         12356      yes  yes
KLJ          Jostons       JKK         15609      Johnsonville  FAD         19876      no   yes
GHT          Barryville    BVC         12398      Unitas        BKD         15609      no   yes
YTY          Arkenstone    AAR         19234      Jenkins       EWQ         12398      no   yes
EWQ          Aberville     ABD         34875      Berryville    ASD         19234      no   yes
VBN          Winston       WEN         12987      Arkass        FDS         34875      yes  yes
MNB          Regulator     REG         11731      Avin          FGH         12987      yes  yes
PLO          Jansen        JHY         19244      Wenston       FGH         12398      no   yes
```
- To escape press ": q"

- To scale the API Server increase the Replica count.
	- NOTE: In vSphere 7.x the CSI driver does not support ReadWriteMany so the PV will only be mounted on a single K8s Node. Any pods scheduled on other Nodes that do not have the PV will fail to schedule successfully.

```shell
 kubectl scale deploy file-api --replicas=3
 	deployment.apps/file-api scaled

k get po -l app=file-api
NAME                             READY   STATUS              RESTARTS   AGE
file-api-675dc68885-5j22b        0/1     ContainerCreating   0          5s
file-api-675dc68885-mx22p        1/1     Running             0          4m13s
file-api-675dc68885-rhnn4        1/1     Running             0          5s

```