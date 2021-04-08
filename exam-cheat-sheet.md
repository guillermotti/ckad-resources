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
# If is not working
# sudo apt-get install bash-completion
# echo "source <(kubectl completion bash)" >> ~/.bashrc
# source ~/.bashrc
# Exam instructions
# man lf_exam
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

Copy marked lines: y
Cut marked lines: d
Paste lines: p or P
:set nonumber + Enter

## kubectl basic help

```sh
k explain po.spec --recursive # pv --recursive
k exec -it nginx -- sh #/bin/bash 
k logs nginx > logs.txt
```

-- /bin/sh -c 'while true; do echo hello; sleep 10;done'

k label po nginx new-label=pepe
k set image deploy nginx nginx=nginx:1.91 //  k set image po nginx nginx=nginx:1.91
k rollout status deploy nginx
k rollout undo deploy nginx --to-revision=2
k rollout history deploy nginx --revision=4
k rollout pause deploy nginx
k scale deploy nginx --replicas=5
k autoscale deploy nginx --min=5 --max=10 --cpu-percent=80

# Create pod
k run NAME --dry-run=client --image=IMAGE --restart=Never --port=PORT --env=KEY=VALUE -o yaml > FILE.yaml
k run nginx --image=nginx --restart=Never -o yaml --dry-run=client --labels=app=v1 -- /bin/sh -c 'echo hello;sleep 3600' > pod.yaml
k run NAME --image=IMAGE --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi'

# Label
k label po nginx key=value
k get po -L app
k get po -l app=v2
k get po -l app=my-app,environment=production
k get po -l environment!=production
k get po -l 'environment in (production,development)'
k label po nginx2 app=v2 --overwrite
k label po nginx1 nginx2 nginx3 app-

# Annotation
k annotate po nginx1 nginx2 nginx3 description='my description'
k describe po nginx1 | grep -i 'annotations'
k annotate po nginx{1..3} description-

# Create deployment
k create deployment NAME --image=IMAGE --dry-run=client -o yaml > FILE.yaml
k create deployment NAME --image=IMAGE --replicas=3 --port=5701
k create deploy NAME --image=IMAGE && k scale deploy/NAME --replicas=N && k label deploy NAME KEY=VALUE

# Create job
k create job NAME --image=IMAGE --dry-run=client -o yaml > FILE.yaml

# Create cronjob
k create cronjob NAME --image=IMAGE --dry-run=client --schedule="* * * * *" -o yaml > FILE.yaml

# Create service & pod
k run nginx --image=nginx --restart=Never --port=80 --expose

# Create service from deploy
k expose deploy foo --port=6262 --target-port=8080
k expose deploy nginx --port=80 --target-port=8000 (--type=ClusterIP,NodePort)

# Create a deployment and then a pod exposing a service to copy the content of the pod to the deployment file
# Copy the content of pod.spec into deploy.spec.template, delete the pod template and run
k create deploy NAME --image=IMAGE $dr > deployment.yaml
k run NAME --image=IMAGE --port=PORT --env=KEY=VALUE --labels=KEY=VALUE --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi' --expose $dr -- /bin/sh -c 'sleep 3600' >> deployment.yaml

# More

k top node
k top pod

k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 10.12.2.15 # To check if the port 80 is accesible
kubectl run busybox --image=busybox --rm -it --restart=Never -- wget -O- 10.1.1.131:80 # The same as before

kubectl run busybox --image=busybox --command --restart=Never -it -- env
kubectl create quota myrq --hard=cpu=1,memory=1G,pods=2 --dry-run=client -o yaml
kubectl get po -A

kubectl rollout pause deploy nginx
kubectl rollout resume deploy nginx

kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'
kubectl create job busybox --image=busybox -- /bin/sh -c 'echo hello;sleep 30;echo world'

kubectl create cronjob busybox --image=busybox --schedule="*/1 * * * *" -- /bin/sh -c 'date; echo Hello from the Kubernetes cluster'
kubectl create cronjob time-limited-job --image=busybox --restart=Never --dry-run=client --schedule="* * * * *" -o yaml -- /bin/sh -c 'date; echo Hello from the Kubernetes cluster' > time-limited-job.yaml
Add cronjob.spec.jobTemplate.spec.activeDeadlineSeconds=17

kubectl create configmap config --from-literal=foo=lala --from-literal=foo2=lolo
echo -e "foo3=lili\nfoo4=lele" > config.txt && kubectl create cm configmap2 --from-file=config.txt
echo -e "var1=val1\n# this is a comment\n\nvar2=val2\n#anothercomment" > config.env && kubectl create cm configmap3 --from-env-file=config.env
echo -e "var3=val3\nvar4=val4" > config4.txt && kubectl create cm configmap4 --from-file=special=config4.txt

kubectl run nginx --image=nginx --restart=Never --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi'

kubectl create secret generic mysecret --from-literal=password=mypass
echo -n admin > username && kubectl create secret generic mysecret2 --from-file=username
kubectl get secret mysecret2 -o jsonpath='{.data.username}{"\n"}' | base64 -d

kubectl get sa -A
kubectl create sa myuser
kubectl run nginx --image=nginx --restart=Never --serviceaccount=myuser -o yaml --dry-run=client > pod.yaml
