# serving-diffusion

This repo creates a serving container for stable diffusion.

**Dockerfile is messy and needs to be cleaned up**

# Setup

1. Clone repo
1. Build container

  ```bash
  docker build . -t gcr.io/{project_id}/stable-diffusion:latest
  ```
1. Push the image

    ```bash
    docker push gcr.io/{project_id}/stable-diffusion:latest
    ```

1. Run container. Needs GPUs.

  ```bash
  docker run -p 80:8080 --gpus all -e AIP_HEALTH_ROUTE=/health -e AIP_HTTP_PORT=8080 -e AIP_PREDICT_ROUTE=/predict gcr.io/{project_id}/stable-diffusion:latest -d
  ```

1. Make a prediction

  ```bash
  python generate_request.py
  curl -X POST -d @request.json -H "Content-Type: application/json; charset=utf-8" localhost/predict >> response.json
  ```
  
 1. Deploy in Vertex AI Endpoints.

    ```bash
    python gcp_deploy.py --project-id <project_id> --bucket gs://<bucket_id>/diffusion-model --image-uri gcr.io/<project_id>/stable-diffusion:latest
    ```

1. Test the endpoint.

    **Txt2img**

    ```python
    from google.cloud import aiplatform

    from google.protobuf import json_format
    from google.protobuf.struct_pb2 import Value

    instances_list = [{"prompt": "A dog wearing a hat"}]
    instances = [json_format.ParseDict(s, Value()) for s in instances_list]

    parameters = {
        'scale' : 7.5, 
        'seed' : 42, 
        'W' : 512, 
        'H' : 512, 
        'ddim_steps' : 50,
        'n_samples' : 2,
        'n_iter' : 2,
        'type' : 'txt2img'
    }
    parameters = json_format.ParseDict(parameters,Value())

    results = endpoint.predict(instances=instances,parameters=parameters)

    ```

    **Img2img**

    ```python
         from google.cloud import aiplatform

    from google.protobuf import json_format
    from google.protobuf.struct_pb2 import Value

    from PIL import Image

    def im_2_b64(image, format='PNG'):
        buff = BytesIO()
        image.save(buff, format=format)
        img_str = base64.b64encode(buff.getvalue())
        return img_str

    image = Image.open("image.png")

    base64_image = im_2_b64(image).decode('utf-8')

    instances_list = [{"prompt": "A dog wearing a hat", 'image' : base64_image}]
    instances = [json_format.ParseDict(s, Value()) for s in instances_list]

    parameters = {
        'strength' : .55,
        'scale' : 7.5, 
        'seed' : 42, 
        'W' : 512, 
        'H' : 512, 
        'ddim_steps' : 50,
        'n_samples' : 2,
        'n_iter' : 2,
        'type' : 'img2img'
    }
    parameters = json_format.ParseDict(parameters,Value())

    results = endpoint.predict(instances=instances,parameters=parameters)

