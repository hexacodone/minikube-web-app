# Minimal Helm Chart Tutorial

## 1. Minikube Setup

```bash
# Installation
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# Installation bestätigen
minikube version
kubectl version --client

# Start mit Podman
minikube start --driver=podman
minikube status
kubectl get nodes
```

## 2. Helm installieren

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Installation bestätigen
helm version
```

## 3. Helm Chart erstellen

```bash
# Chart erstellen
helm create web-app
cd web-app

# Unnötige Dateien entfernen
rm -rf templates/tests templates/hpa.yaml templates/serviceaccount.yaml templates/NOTES.txt
```

## 4. values.yaml anpassen

```yaml
# values.yaml
replicaCount: 1

image:
  repository: nginx
  tag: alpine
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  host: web-app.local

# Custom HTML Content
htmlContent: |
  <!DOCTYPE html>
  <html>
  <head>
      <meta charset="UTF-8">
      <title>Helm Nginx App</title>
  </head>
  <body>
      <h1>🚀 Helm Chart funktioniert!</h1>
      <p>Custom HTML aus ConfigMap</p>
  </body>
  </html>
```

## 5. Templates anpassen

### ConfigMap Template
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "web-app.fullname" . }}-html
data:
  index.html: |
{{ .Values.htmlContent | indent 4 }}
```

### Deployment Template
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "web-app.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "web-app.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "web-app.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html-volume
          configMap:
            name: {{ include "web-app.fullname" . }}-html
```

### Ingress Template
```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "web-app.fullname" . }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "web-app.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

## 6. Helm Chart deployen

```bash
# Chart installieren
helm install web-app .

# Installation bestätigen
helm status web-app
kubectl get pods
kubectl get configmaps
kubectl get services

# Testen
kubectl port-forward service/web-app 8080:80
curl http://localhost:8080
```

## 7. Ingress aktivieren

```bash
# Ingress Controller aktivieren
minikube addons enable ingress

# Ingress-Status prüfen
kubectl get pods -n ingress-nginx

# Ingress in values.yaml aktivieren
# Ändern: ingress.enabled: true

# Chart upgraden
helm upgrade web-app . --set ingress.enabled=true

# Ingress bestätigen
kubectl get ingress
kubectl describe ingress web-app

# Port-Forward für externen Zugriff
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80 --address 0.0.0.0

# Schritt für Schritt:

# Ingress-Controller läuft im Cluster (Port 80)
# Port-Forward mappt VM-Port 8080 → Cluster-Port 80
# --address 0.0.0.0 macht Port auf allen VM-Interfaces verfügbar
# Get-Content C:\Windows\System32\drivers\etc\hosts
# Netzwerk-Flow:
# Windows Browser → Proxmox VM:8080 → Minikube Cluster → Ingress Controller → web-app Pod
```


## 8. Testen

```bash
# Lokal testen
curl -H "Host: web-app.local" http://localhost:8080

# Windows hosts-Datei: IP web-app.local
# Browser: http://web-app.local:8080
```

## Aufräumen

```bash
# 1. Helm Release löschen
helm uninstall web-app

# 2. Alle ConfigMaps löschen
kubectl delete configmap --all

# 3. Alle Kubernetes-Ressourcen löschen
kubectl delete all --all
kubectl delete ingress --all

# 4. Port-Forward stoppen
pkill -f "kubectl port-forward"

# 5. Zustand prüfen (sollte leer sein)
kubectl get all
kubectl get configmaps
kubectl get ingress

# 6. Minikube stoppen (optional)
minikube stop

# 7. Minikube komplett löschen (falls gewünscht)
minikube delete
```

## Lernziele erreicht ✅

1. **Minikube installiert** und gestartet
2. **Nginx mit Helm Chart** deployed
3. **Custom HTML aus ConfigMap** gemountet
4. **Ingress konfiguriert** und funktionsfähig