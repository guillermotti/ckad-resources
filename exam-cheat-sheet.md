# Exam Cheat Sheet

## Set environment

```sh
alias k=kubectl
alias kubens='kubectl config set-context --current --namespace '
export dr="--dry-run=client -o yaml"
export gp="--grace-period=0 --force"
export an="--all-namespaces"
export sl="--show-labels"
source <(kubectl completion bash)
complete -F __start_kubectl k
```

## Set vim config

```sh
vim ~/.vimrc
```

```vim
set tabstop=2
set softtabstop=2
set expandtab
set shiftwidth=2
set number
```

## Vim basic help

Copy lines: y
Cut lines: d
Paste lines: p
:set nonumber + Enter

## kubectl basic help

```sh
k top node
k top pod
k get all
k explain po.spec --recursive
k exec nginx -- sh -c 'while true; do echo hello; sleep 2;done'
k exec secret-handler -- find /tmp/secret2
k logs nginx > logs.txt
```

# Pod
k run NAME --image=IMAGE $dr
k run tmp --restart=Never --rm -it --image=nginx:alpine -- curl -m 5 IP:PORT
k run tmp --restart=Never --rm -it --image=busybox -- wget -O- IP:PORT
k run busybox --restart=Never --rm -it --image=busybox -- sh -c env
k set image po nginx nginx=nginx:1.91
## Arguments
--port=PORT 
--env=KEY=VALUE 
--labels=KEY=VALUE
--requests='cpu=100m,memory=256Mi' 
--limits='cpu=200m,memory=512Mi'
--serviceaccount=SA
--restart=Never 
--rm 
-it
-- sh -c 'echo hello;sleep 3600'
--expose # Create pod and service
> FILE.yaml

# Configmap
k create configmap config --from-literal=foo=lala --from-literal=foo2=lolo
echo -e "foo3=lili\nfoo4=lele" > config.txt && k create cm configmap2 --from-file=config.txt
echo -e "var1=val1\n# this is a comment\n\nvar2=val2\n#anothercomment" > config.env && k create cm configmap3 --from-env-file=config.env
echo -e "var3=val3\nvar4=val4" > config4.txt && k create cm configmap4 --from-file=special=config4.txt

# Secret
k create secret generic mysecret --from-literal=password=mypass
echo -n admin > username && k create secret generic mysecret2 --from-file=username
k get secret mysecret2 -o jsonpath='{.data.username}{"\n"}' | base64 -d

# Label
k label po nginx key=value
k label po nginx key=value --overwrite
k label po nginx1 nginx2 nginx3 app-
k get po -L app
k get po -l app=v2
k get po -l app=my-app,environment=production
k get po -l environment!=production
k get po -l 'environment in (production,development)'

# Annotation
k annotate po nginx1 nginx2 nginx3 description='my description'
k annotate po nginx{1..3} description-
k describe po nginx | grep -i 'annotations'

# Deployment
k create deploy NAME --image=IMAGE $dr
k create deploy NAME --image=IMAGE && k label deploy NAME KEY=VALUE

## Arguments
--replicas=N
--port=PORT
> FILE.yaml

## Manage deployments
k set image deploy nginx nginx=nginx:1.91
k rollout status deploy nginx
k rollout undo deploy nginx --to-revision=2
k rollout history deploy nginx --revision=4
k rollout pause deploy nginx
k rollout resume deploy nginx
k scale deploy nginx --replicas=5
k autoscale deploy nginx --min=5 --max=10 --cpu-percent=80

# Job
k create job NAME --image=IMAGE $dr -- sh -c 'echo hello;sleep 3600' > FILE.yaml

# Cronjob
k create cronjob NAME --image=IMAGE --schedule="* * * * *" $dr -- sh -c 'echo hello;sleep 3600' > FILE.yaml

# Service
k create service clusterip NAME --tcp PORT:TARGET_PORT $dr > FILE.yaml

# Service from po/deploy
k expose po/deploy NAME --name=SVC_NAME --port=PORT --target-port=PORT (--type=ClusterIP,NodePort)

# Create a deployment and then a pod exposing a service to copy the content of the pod to the deployment file
# Copy the content of pod.spec into deploy.spec.template, delete the pod template and run
k create deploy NAME --image=IMAGE $dr > deployment.yaml
k run NAME --image=IMAGE --port=PORT --env=KEY=VALUE --labels=KEY=VALUE --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi' --expose $dr -- sh -c 'sleep 3600' >> deployment.yaml

# Quota
kubectl create quota myrq --hard=cpu=1,memory=1G,pods=2 $dr

# If bash-completion is not working
# sudo apt-get install bash-completion
# echo "source <(kubectl completion bash)" >> ~/.bashrc
# source ~/.bashrc
# Exam instructions
# man lf_exam