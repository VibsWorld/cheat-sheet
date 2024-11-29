## Port Forward Example
```
kubectl port-forward pods/<podname> 8080:80 -n <namespace>
kubectl port-forward svc/<service_name> 8080:80 -n <namespace>
kubectl port-forward pods/<get_name_from_above_command> <Your_Port_Localhost>:<Listening_port_of_container> -n <NameSpace>
#Example
kubectl port-forward pods/padauk-5b8c899668-8tx7x 8080:80 -n cd-sandboxconfig
```
### Port Forward Script Example (Windows Batch file)
```
title Aspen Server
set nameSpace=cd-sandboxconfig
set podRealName=aspen
set portForward=8080
set podPort=80
set urlScheme=http
cls
REM ********Now Script Starts*********************
kubectl -n %nameSpace% get pods --sort-by=.metadata.creationTimestamp --selector=app.kubernetes.io/name=%podRealName% -o jsonpath="{.items[0].metadata.name}" > temp.txt
set /p podName=<temp.txt
echo Pod instance found is %podName%
start "Aspen Port Forward on %portForward% for %podName%" kubectl port-forward %podName% %portForward%:%podPort% -n %nameSpace%
del temp.txt
cls
ping -n 2 127.0.0.1>nul
start %urlScheme%://localhost:%portForward%/api/accounts
```

## Kubectl common commands
```
kubectl config view 
kubectl config get-contexts
kubectl config current-context
kubectl config use-context dev-aks
kubectl describe pods padauk --namespace cd-sandboxconfig
```
## Kubectl Pods Specific
```
--Get name of latest pods
kubectl -n cd-sandboxconfig get pods --selector=app.kubernetes.io/name=padauk -o name
kubectl -n cd-sandboxconfig get pods --sort-by=.metadata.creationTimestamp --selector=app.kubernetes.io/name=olive -o jsonpath="{.items[0].metadata.name}"
```
