# Wowza Streaming Engine Deployment and Service

This project sets up a Wowza Streaming Engine containerized deployment using Kubernetes, along with a service to expose it via NodePort. It includes a deployment configuration for the Wowza Streaming Engine, a config map for VOD applications, and a service for accessing the streaming engine.

## Overview

### 1. **Deployment**
The `Deployment` is configured to deploy a single replica of the Wowza Streaming Engine container. The container runs the `wowzamedia/wowza-streaming-engine-linux:latest` image and executes a custom entry point script (`/sbin/entrypoint.sh`). It also mounts the necessary configuration file for VOD (Video On Demand) and the content directory for media storage.

### 2. **Service**
A Kubernetes `Service` of type `NodePort` exposes the Wowza Streaming Engine to external traffic. Two ports are exposed: one for the streaming (port 1935) and one for the Wowza Manager (port 8088).

## Kubernetes Resources

### Deployment Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wowza-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wowza
  template:
    metadata:
      labels:
        app: wowza
    spec:
      containers:
      - name: wowza
        image: wowzamedia/wowza-streaming-engine-linux:latest
        command: ["/bin/sh", "-c", "/sbin/entrypoint.sh"]
        ports:
        - containerPort: 1935
        - containerPort: 8088
        env:
        - name: WSE_LIC
          value: "ET1B4-TTEpx-CGJHA-zHHvy-63KzG-P6RAp-47UeyP7mNvWp"  # License key
        - name: WSE_MGR_USER
          value: "admin"  # Username for Wowza Manager
        - name: WSE_MGR_PASS
          value: "password"  # Password for Wowza Manager
        volumeMounts:
        - name: wowza-vod-application-configmap
          mountPath: /usr/local/WowzaStreamingEngine/conf/vod/Application.xml
          subPath: Application.xml  # Mount VOD application config file
        - name: wowza-content-volume
          mountPath: /usr/local/WowzaStreamingEngine/content  # Media content path
      volumes:
      - name: wowza-vod-application-configmap
        configMap:
          name: wowza-vod-application-configmap  # ConfigMap for VOD application config
      - name: wowza-content-volume
        hostPath:
          path: /tmp  # Set the media storage directory
          type: Directory
```

### Service Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wowza-service
spec:
  type: NodePort
  selector:
    app: wowza
  ports:
  - name: port-1935
    protocol: TCP
    port: 1935
    targetPort: 1935
    nodePort: 30035  # Expose port 1935 for streaming protocol
  - name: port-8088
    protocol: TCP
    port: 8088
    targetPort: 8088
    nodePort: 30088  # Expose port 8088 for Wowza Manager
```

## Environment Variables

- `WSE_LIC`: License key for Wowza Streaming Engine.
- `WSE_MGR_USER`: Username for the Wowza Manager UI.
- `WSE_MGR_PASS`: Password for the Wowza Manager UI.

## Volumes

1. **Wowza VOD Application ConfigMap**: The configuration file for Video On Demand (VOD) is mounted into the container from a Kubernetes `ConfigMap`. This file is expected to be located at `/usr/local/WowzaStreamingEngine/conf/vod/Application.xml` inside the container.
   
2. **Wowza Content Volume**: The media content for Wowza is mounted from a host path (ex. `/tmp`) to the container's `/usr/local/WowzaStreamingEngine/content` directory.

## Exposing Ports

- **Port 1935 (Streaming protocols)**: Exposes the streaming services. This is mapped to `NodePort 30035`.
- **Port 8088 (Wowza Manager)**: Exposes the Wowza Manager UI for managing streams. This is mapped to `NodePort 30088`.

You can access these services on the node's IP, like so:

- Streaming: `http://<node-ip>:30035`
- Wowza Manager: `http://<node-ip>:30088`

## How to Use

1. **Deploy the Resources**:
   Run the following command to deploy the resources:

   ```bash
   kubectl apply -f deployment.yaml
   kubectl apply -f service.yaml
   ```

2. **Access the Services**:
   - Use the node's IP to access the streaming services on port 30035.
   - Use the node's IP to access the Wowza Manager on port 30088.

3. **Edit Configuration**:
   If you need to modify the VOD configuration, you can update the `Application.xml` file in the `ConfigMap` and apply the changes:

   ```bash
   kubectl create configmap wowza-vod-application-configmap --from-file=Application.xml -o yaml --dry-run | kubectl apply -f -
   ```

## Additional Notes

- Ensure that the `/tmp` directory on your host machine contains the required media content for Wowza to serve.
- The `ConfigMap` and media content volumes can be updated independently without needing to restart the Wowza container.
