https://www.librechat.ai/toolkit/creds_generator

kubectl apply -f secrets.yaml

# Instalación básica

helm install librechat oci://ghcr.io/danny-avila/librechat-chart/librechat

# O con archivo de valores personalizado

helm install librechat oci://ghcr.io/danny-avila/librechat-chart/librechat --values values.yaml

kubectl get pods
kubectl get services

kubectl port-forward -n default
librechat-librechat-...
3080:3080

helm uninstall librechat

Cline
https://cline.bot/pricing

CONTINUE DEV WITH VSCODE
https://www.youtube.com/watch?v=he0_W5iCv-I
