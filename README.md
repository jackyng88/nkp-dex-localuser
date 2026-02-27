# Adding a local user to an NKP Cluster via Dex


## Prequisites:
- kubectl CLI installed on Linux jumphost/jumpbox
- Connectivity to the NKP/Kubernetes cluster via kubeconfig file

## Steps:
- Run the following command on a Linux jumphost or system that has access to the NKP/Kubernetes cluster via Kubeconfig. 
```kubectl get cm -n kommander | grep dex```
- It should show a file named dex-kommander-overrides. This configmap has override values for dex.
- Run the following command. Replace password123 with whatever password you would like. It's a Bcrypt encrypted hash that Dex accepts and requires.
```echo password123 | htpasswd -BinC 10 admin | cut -d: -f2```
- You will get a value like "$2y$10$unM8gxYTH.1EdxR1.:
ThxYOI5zhFvOV9nCwt093KEPgiEI2EWujkfO". Save this value.

- We will now need to update the `dex-kommander-overrides` configmap file. Run the command:
```kubectl edit configmap dex-kommander-overrides -n kommander```

- Once in the text editor, you should see in the yaml the section `data.values.yaml.config`. It should have an issuer section by default. We will need to edit it to include a new `staticPasswords` section with a few subsections like the following:

```
data:
  values.yaml: |
    config:
      issuer: https://11.11.111.111/dex
      staticPasswords:
      - email: "jackyng"
        hash: "$2y$10$LzC3Y4w7fVqvqWzcftdFdeIQcWtrVNRvTbd25Vvdru/rmbAs54Eaq"
        username: "Jacky Ng"
        userID: "1234-555-666"
```

- Important things to note:
- - `email` will be the login value or userid. It's usually a password but it can be a regular string.
- - `hash` is the Bcrypt encrypted hash value we created earlier.
- - `username` not particularly important but it will be the display name in the NKP web console.
- - `userID` just a random string value you can create to be a unique identifier for the user.

- Remember to save the edited YAML file and exit.

- Next we will have to assign one of the pre-existing admin cluster role bindings to the user. You can always create more roles or to assign different roles. Important: Update the `--user=` flag with a value that is the same value as `email` in the previous step. 

```
kubectl create clusterrolebinding local-admin-binding \
  --clusterrole=cluster-admin \
  --user=jackyng
```

- The dex appdeployment may not reconcile by itself so we may need to force a reconciliation via the following command:
```
kubectl rollout restart deployment dex -n kommander
```

- Check the status and make sure it says 1/1 Ready:

```
kubectl get deployment dex -n kommander
```

```
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
dex    1/1     1            1           36d
```

- To make sure that the new local user connector configuration is working, run the following:

```
kubectl logs -n kommander -l app.kubernetes.io/name=dex -f
```

- You should see the following text indicating that the configuration has succeeded:

```
time=2026-02-27T16:39:21.271Z level=INFO msg="config connector: local passwords enabled"
```

- Log into your NKP web console and login with the newly created local user
- - `Username` will be the value of `email` we added to the configmap earlier.
- - `Password` will not be the Bcrypt hash but the value we used to generate the hash. In my previous example I used `password123`.

- You have successfully added a new local user via Dex.
