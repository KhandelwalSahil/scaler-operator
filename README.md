# scaler-operator
A sample operator which can scale pods based on time window which can be useful to manage peek traffic cycles.

## Description
There are 3 main components for any operator to work:
1. Custom Resource Definition (CRDs): Used to extend kubernetes with new behavior or functionalities
2. Custom Resource (CRs): A custom resource is an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. It represents a customization of a particular Kubernetes installation.
3. Controller: Controller will observe your objects, analyze the changes compared to the cluster's current state, and apply actions that transition the cluster into the new desired state.

### Prerequisites
- go
- docker
- Minikube or Kind (or any other kubernetes env)
- kubectl
- Operator-Framework (Operator-SDK) (Installation Steps: https://sdk.operatorframework.io/docs/installation/)

### Steps to follow:

**Create scaler-operator directory and go inside it**
```sh
mkdir scaler-operator && cd scaler-operator
```

**Initializing Operator Template Code using Operator-SDK**
```sh
operator-sdk init --plugins go/v4 --owner "sahil khandelwal" --repo github.com/khandelwalsahil/scaler-operator
```

**Creating API for kind Scaler**
```sh
operator-sdk create api --kind Scaler --group api --version v1alpha1
```
**Give y for Create Resource and Controller**
Resource/API: Path where we will define our custom resource object
Controller: Path where we will write our logic to interact with these objects for reconciliation

**Create template for CRD file**
```sh
make manifests
```

**Paths for few important files to work on**
1. CRD: config/crd/bases/
2. CR: config/samples/
3. Controller: internal/controller/scaler_controller.go
4. API Object: api/v1alpha1/scaler_types.go

**Defining Custom Resource**
```sh
apiVersion: api.my.domain/v1alpha1
kind: Scaler
metadata:
  labels:
    app.kubernetes.io/name: scaler
    app.kubernetes.io/instance: scaler-sample
    app.kubernetes.io/part-of: scaler-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: scaler-operator
  name: scaler-sample
spec:
  start: 5
  end: 10
  replicas: 5
  deployments:
    - name: abc
      namespace: default
```

**Defining API Object (As per above CR, add fields under spec)"**
```sh
type ScalerSpec struct {
	
	Start      int              `json:"start"`
	End        int              `json:"end"`
	Replicas   int32            `json:"replicas"`
	Deployment []NameSpacedName `json:"deployments"`
}

type NameSpacedName struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
}
```

**Create CRD (config/crd/bases/api.my.domain_scalers.yaml)**
```sh
make Manifests
```

**Write Reconcile Logic for Controller**
```sh
func (r *ScalerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := logger.WithValues("Request.Namespace", req.Namespace, "Request.Name", req.Name)

	log.Info("Reconcile called")
	scaler := &apiv1alpha1.Scaler{}

	err := r.Get(ctx, req.NamespacedName, scaler)
	if err != nil {
		if apierrors.IsNotFound(err) {
			log.Info("Scaler resource not found. Ignoring since object must be deleted.")
			return ctrl.Result{}, nil
		}
		log.Error(err, "Failed")
		return ctrl.Result{}, err
	}

	startTime := scaler.Spec.Start
	endTime := scaler.Spec.End

	// current time in UTC
	currentHour := time.Now().UTC().Hour()
	log.Info(fmt.Sprintf("current time in hour : %d\n", currentHour))

	if currentHour >= startTime && currentHour <= endTime {

		if err = scaleDeployment(scaler, r, ctx, int32(scaler.Spec.Replicas)); err != nil {
			return ctrl.Result{}, err
		}
	}

	return ctrl.Result{RequeueAfter: time.Duration(30 * time.Second)}, nil
}

func scaleDeployment(scaler *apiv1alpha1.Scaler, r *ScalerReconciler, ctx context.Context, replicas int32) error {
	for _, deploy := range scaler.Spec.Deployment {
		dep := &v1.Deployment{}
		err := r.Get(ctx, types.NamespacedName{
			Namespace: deploy.Namespace,
			Name:      deploy.Name,
		}, dep)
		if err != nil {
			return err
		}

		if dep.Spec.Replicas != &replicas {
			dep.Spec.Replicas = &replicas
			err := r.Update(ctx, dep)
			if err != nil {
				scaler.Status.Status = apiv1alpha1.FAILED
				return err
			}
			scaler.Status.Status = apiv1alpha1.SUCCESS
			err = r.Status().Update(ctx, scaler)
			if err != nil {
				return err
			}
		}
	}
	return nil
}
```

**Now we can start our cluster and execute our changes**

**Start Cluster (I am using minikube)**
```sh
minikube start
```

**Apply CRD**
```sh
kubectl apply -f config/crd/bases/api.my.domain_scalers.yaml
```

**Deploy nginx (It will start 2 pods)**
```sh
 kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

**Start Operator**
```sh
make run
```

**Apply CR to scale pods to 5**
```sh
kubectl apply -f config/samples/api_v1alpha1_scaler.yaml
```

It should return 5 pods
```sh
kubectl get pods
```

**This was the sample operator to just get acquainted with the operator, CR and CRDs and related concepts**
