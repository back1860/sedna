# Using Federated Learning Job in Surface Defect Detection Scenario
This case introduces how to use federated learning job in surface defect detection scenario.
In the safety surface defect detection, data is scattered in different places (such as server node, camera or others) and cannot be aggregated due to data privacy and bandwidth. As a result, we cannot use all the data for training.
Using Federated Learning, we can solve the problem. Each place uses its own data for model training ,uploads the weight to the cloud for aggregation, and obtains the aggregation result for model update.


## Surface Defect Detection Experiment
> Assume that there are two edge nodes and a cloud node. Data on the edge nodes cannot be migrated to the cloud due to privacy issues.
> Base on this scenario, we will demonstrate the surface inspection.

### Prepare Nodes
```
CLOUD_NODE="cloud-node-name"
EDGE1_NODE="edge1-node-name"
EDGE2_NODE="edge2-node-name"
```

### Install Sedna

Follow the [Sedna installation document](/docs/setup/install.md) to install Sedna.
 
### Prepare Dataset

Download [dataset](https://github.com/abin24/Magnetic-tile-defect-datasets.) and the [label file](/examples/federated_learning/surface_defect_detection/data/1.txt) to `/data` of ```EDGE1_NODE```.  
```
mkdir -p /data
cd /data
git clone https://github.com/abin24/Magnetic-tile-defect-datasets..git Magnetic-tile-defect-datasets
curl -o 1.txt https://raw.githubusercontent.com/kubeedge/sedna/main/examples/federated_learning/surface_defect_detection/data/1.txt
```

Download [dataset](https://github.com/abin24/Magnetic-tile-defect-datasets.) and the [label file](/examples/federated_learning/surface_defect_detection/data/2.txt) to `/data` of ```EDGE2_NODE```.
```
mkdir -p /data
cd /data
git clone https://github.com/abin24/Magnetic-tile-defect-datasets..git Magnetic-tile-defect-datasets
curl -o 2.txt https://raw.githubusercontent.com/kubeedge/sedna/main/examples/federated_learning/surface_defect_detection/data/2.txt
```

### Prepare Images
This example uses these images:
1. aggregation worker: ```kubeedge/sedna-example-federated-learning-surface-defect-detection-aggregation:v0.3.0```
2. train worker: ```kubeedge/sedna-example-federated-learning-surface-defect-detection-train:v0.3.0```

These images are generated by the script [build_images.sh](/examples/build_image.sh).

### Create Federated Learning Job 

#### Create Dataset

create dataset for `$EDGE1_NODE`
```
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Dataset
metadata:
  name: "edge1-surface-defect-detection-dataset"
spec:
  url: "/data/1.txt"
  format: "txt"
  nodeName: $EDGE1_NODE
EOF
```

create dataset for `$EDGE2_NODE`
```
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Dataset
metadata:
  name: "edge2-surface-defect-detection-dataset"
spec:
  url: "/data/2.txt"
  format: "txt"
  nodeName: $EDGE2_NODE
EOF
```

#### Create Model

create the directory `/model` in the host of `$EDGE1_NODE`
```
mkdir /model
```
create the directory `/model` in the host of `$EDGE2_NODE`
```
mkdir /model
```

create model
```
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Model
metadata:
  name: "surface-defect-detection-model"
spec:
  url: "/model"
  format: "pb"
EOF
```

#### Start Federated Learning Job

```
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: FederatedLearningJob
metadata:
  name: surface-defect-detection
spec:
  aggregationWorker:
    model:
      name: "surface-defect-detection-model"
    template:
      spec:
        nodeName: $CLOUD_NODE
        containers:
          - image: kubeedge/sedna-example-federated-learning-surface-defect-detection-aggregation:v0.3.0
            name:  agg-worker
            imagePullPolicy: IfNotPresent
            env: # user defined environments
              - name: "exit_round"
                value: "3"
            resources:  # user defined resources
              limits:
                memory: 2Gi
  trainingWorkers:
    - dataset:
        name: "edge1-surface-defect-detection-dataset"
      template:
        spec:
          nodeName: $EDGE1_NODE
          containers:
            - image: kubeedge/sedna-example-federated-learning-surface-defect-detection-train:v0.3.0
              name:  train-worker
              imagePullPolicy: IfNotPresent
              env:  # user defined environments
                - name: "batch_size"
                  value: "32"
                - name: "learning_rate"
                  value: "0.001"
                - name: "epochs"
                  value: "2"
              resources:  # user defined resources
                limits:
                  memory: 2Gi
    - dataset:
        name: "edge2-surface-defect-detection-dataset"
      template:
        spec:
          nodeName: $EDGE2_NODE
          containers:
            - image: kubeedge/sedna-example-federated-learning-surface-defect-detection-train:v0.3.0
              name:  train-worker
              imagePullPolicy: IfNotPresent
              env:  # user defined environments
                - name: "batch_size"
                  value: "32"
                - name: "learning_rate"
                  value: "0.001"
                - name: "epochs"
                  value: "2"
              resources:  # user defined resources
                limits:
                  memory: 2Gi
EOF
```

### Check Federated Learning Status

```
kubectl get federatedlearningjob surface-defect-detection
```

### Check Federated Learning Train Result
After the job completed, you will find the model generated on the directory `/model` in `$EDGE1_NODE` and `$EDGE2_NODE`.