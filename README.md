# k8s-keystone-auth

Proof-Of-Concept : Kubernetes webhook authentication and authorization for OpenStack Keystone

Steps to use this webook with Kubernetes

- Save the following into webhook.kubeconfig.
```
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://localhost:8443/webhook
  name: webhook
contexts:
- context:
    cluster: webhook
    user: webhook
  name: webhook
current-context: webhook
kind: Config
preferences: {}
users:
- name: webhook
```
- Copy the examples/policy.json and edit it to your needs.
- Add the following flags to your Kubernetes api server.
  * `--authentication-token-webhook-config-file=/path/to/your/webhook.kubeconfig`
  * `--authorization-mode=Webhook --authorization-webhook-config-file=/path/to/your/webhook.kubeconfig`
- Start webhook process with the following flags
  * `--tls-cert-file /var/run/kubernetes/serving-kube-apiserver.crt`
  * `--tls-private-key-file /var/run/kubernetes/serving-kube-apiserver.key`
  * `--keystone-policy-file examples/policy.json`
- Run `openstack token issue` to generate a token
- Run `kubectl --token $TOKEN get po` or `curl -k -v -XGET  -H "Accept: application/json" -H "Authorization: Bearer $TOKEN" https://localhost:6443/api/v1/namespaces/default/pods`

More details about Kubernetes Authentication Webhook using Bearer Tokens is at :
https://kubernetes.io/docs/admin/authentication/#webhook-token-authentication

and the Authorization Webhook is at:
https://kubernetes.io/docs/admin/authorization/webhook/

Tips:

- You can directly test the webhook with
```
cat << EOF | curl -kvs -XPOST -d @- https://localhost:8443/webhook | python -mjson.tool
{
	"apiVersion": "authentication.k8s.io/v1beta1",
	"kind": "TokenReview",
	"metadata": {
		"creationTimestamp": null
	},
	"spec": {
		"token": "$TOKEN"
	}
}
EOF

cat << EOF | curl -kvs -XPOST -d @- https://localhost:8443/webhook | python -mjson.tool
{
	"apiVersion": "authorization.k8s.io/v1beta1",
	"kind": "SubjectAccessReview",
	"spec": {
		"resourceAttributes": {
			"namespace": "kittensandponies",
			"verb": "get",
			"group": "unicorn.example.org",
			"resource": "pods"
		},
		"user": "jane",
		"group": [
			"group1",
			"group2"
		]
	}
}
EOF
```
