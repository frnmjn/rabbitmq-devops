# Install kind

<https://kind.sigs.k8s.io/docs/user/quick-start/#installation>

# Create a k8s cluster

kind create cluster --name rabbitmq-workshop

# Install rabbitmq plugin

<https://krew.sigs.k8s.io/docs/user-guide/setup/install/>

<https://www.rabbitmq.com/kubernetes/operator/kubectl-plugin.html#install>

# Install operator

kubectl rabbitmq install-cluster-operator

# Create cluster

kubectl rabbitmq create rabbitmq-workshop --replicas 3

# Show credential

kubectl rabbitmq secrets rabbitmq-workshop

# Port forward admin dashboard

kubectl rabbitmq manage rabbitmq-workshop
