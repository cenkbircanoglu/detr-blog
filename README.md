## Summary of the paper
"End-to-End Object Detection with Transformers" describes a computer vision methodology that utilizes transformer-based architectures for direct object detection, eliminating the requirement for conventional techniques such as region proposal networks.

The approach involves adapting transformer architecture, originally designed for natural language processing, to address object detection tasks. Transformers, valued for their proficiency in capturing extensive dependencies in sequential data, demonstrate suitability for tasks extending beyond language processing.

The architecture adopts a typical encoder-decoder structure prevalent in sequence-to-sequence tasks. The encoder handles input data (image features), extracting relevant information, while the decoder generates predictions like bounding boxes and class labels using the encoded details.

As transformers operate on 2D dimensions, a flattening operation is necessary for processing image data within the model.

Positional embeddings are incorporated into the model to convey details about the spatial arrangement of elements in the input data.
Positional embeddings are generated to augment the transformer's capability to comprehend spatial relationships within the image. There are several possible positional embeddings generation such as sine based, learnable etc.

The model uses object queries to focus on specific regions of the input image implicitly. Object queries act as learnable parameters that guide the attention mechanism to relevant parts of the image, contributing to accurate object localization.

The model is trained end-to-end, optimizing parameters to minimize the difference between predicted and ground truth bounding boxes and class labels.

Unlike traditional object detection methods, the model eliminates the need for non-maximum suppression (NMS) during post-processing.
The model can directly output multiple bounding boxes for a single object, addressing the need for selecting the most confident prediction.

In this notebook, I delve into the original implementation of the DEtection TRansformer (DETR) model from the "End-to-End Object Detection with Transformers" paper, accessible on [GitHub](https://github.com/facebookresearch/detr). To enhance the model's comprehensibility, I've organized it into distinct components:

- Batch Creation
- Feature Map Extraction from CNN
- Addition of Positional Embeddings to Feature Maps
- Acquisition of High-Level Activation Maps
- Transformer Encoder (Single Layer/Multi-Layer)
- Transformer Decoder (Single Layer/Multi-Layer)
- MLP Layer for BBox Prediction
- Linear Layer for Class Prediction

Additionally, I have excluded the aux_loss from the original implementation to simplify the code. Towards the end, I assembled the DETR model using the modified modules. It's important to note that this code is not written entirely from scratch; rather, modifications have been made for improved clarity.

As a note, object detection set prediction loss which makes this architrecture works is another important part. However, to focus on one part, I didn't add the implementation of the loss function to here.


```python
import warnings
warnings.filterwarnings("ignore")
```


```python
%%capture
! pip install ipyplot pillow torch torchvision
```


```python
import math

import torch
import torch.nn as nn
import torch.nn.functional as F
```


```python
# Read two images from the disk
from PIL import Image
microwave = Image.open('./images/microwave.jpeg')
car = Image.open('./images/car.jpeg')
print(f'Dimensions of the microwave image: {microwave.size} and car image: {car.size}')
```

    Dimensions of the microwave image: (425, 639) and car image: (640, 622)



```python
# Convert the pillow images to torch tensor
import torchvision.transforms as T
microwave_tensor = T.PILToTensor()(microwave)
car_tensor = T.PILToTensor()(car)
```


```python
# Create the tensor list
image_list = [microwave_tensor, car_tensor]
```

### Create Batches
<div style="text-align:center">
    <img src='https://raw.githubusercontent.com/cenkbircanoglu/detr-blog/main/images/input_batch.png' width=70% height=50%/>,
</div>
As mentioned in the paper with the preceding sentence, the initial step involves determining the maximum height and width among the images in the batch. Subsequently, apply zero-padding to the original images to ensure uniform dimensions. Additionally, generate a mask tensor indicating whether each image has undergone zero-padding or not. 


```python
def resize_images_and_create_masks(batch):
    # Find the maximum Height and width
    max_height, max_width = max(image.shape[1] for image in batch), max(image.shape[2] for image in batch)
    print(f'Max Width and Height value in the batch: {max_width}, {max_height}')

    # Initialise a zero tensor with the dimension of desired batch
    padded_batch = torch.zeros((len(batch), 3, max_height, max_width), dtype=batch[0].dtype)
    # Initialise a one-tensor with the dimension of desired batch
    mask_batch = torch.ones((len(batch), max_height, max_width), dtype=torch.bool)

    # Copy the images to padded_batch and set the mask 0 for the pixels which is not in the original image
    for image, pad_image, mask in zip(batch, padded_batch, mask_batch):
        pad_image[:, :image.shape[1], :image.shape[2]].copy_(image)
        mask[:image.shape[1], :image.shape[2]] = False

    return padded_batch, mask_batch
```


```python
# Calculate the padded_batch and mask_batch from image_list
padded_batch, mask_batch = resize_images_and_create_masks(image_list)
print(f'Shape of the Padded Batch {list(padded_batch.shape)}, Shape of the masks: {list(mask_batch.shape)}')
```

    Max Width and Height value in the batch: 640, 639
    Shape of the Padded Batch [2, 3, 639, 640], Shape of the masks: [2, 639, 640]


As it is shown here, the images are 0 padded according to the new image dimensions
<div style="text-align:center">
    <img src='https://raw.githubusercontent.com/cenkbircanoglu/detr-blog/main/images/batch.png' width=70% height=50%/>
</div>

## Overall Architecture

The overall architecture of DETR model contains:

- Image Encoder (Backbone): The input image passes through a pre-trained CNN model (ResNet50), without dense and pooling layers, to extract feature maps. These feature maps capture hierarchical information about the input image, from low-level to high-level features.

- Positional Encoding: The feature maps are augmented with positional embeddings to provide spatial information to the model.
The positional embeddings are added to the feature maps, allowing the model to take into account the absolute position of each element.

- Object Queries: A set of learnable object queries is introduced. Each query represents a potential object in the image. 

- Transformer Encoder: The augmented feature maps, masks, and positional encodings,  are passed through a transformer encoder.
The transformer encoder processes the input in a self-attention mechanism, allowing the model to capture global dependencies and relationships.

- Decoder: The output of the transformer encoder is fed into the decoder with masks, positional embeddings, object queries, target embeddings and query embeddings. The decoder consists of two transformer decoder layers. Each decoder layer refines the object queries and generates predictions for the bounding boxes and class labels.

- Class Predictions: A set of probability scores indicating the presence of each class for each object query.
- Box Predictions: Predictions for the bounding boxes (coordinates) of the detected objects.

<div style="text-align:center">
    <img src='https://raw.githubusercontent.com/cenkbircanoglu/detr-blog/main/images/architecture.png' width=70% height=50%/>,
</div>

### Extract Feature Maps
<div style="text-align:center">
    <img src='https://raw.githubusercontent.com/cenkbircanoglu/detr-blog/main/images/backbone.png' width=70% height=50%/>,
</div>
In the paper, the authors mentioned exactly "the backbone is with ImageNet-pretrained ResNet model [15] from torchvision with frozen batchnorm layers."

The original implementation contains extracting the intermediate block outputs also. And 


```python
# Initialise Backbone model from Pretrained ResNet50
import torchvision
from torchvision.ops import FrozenBatchNorm2d

class Backbone(nn.Module):
    def __init__(self):
        super().__init__()
        # get the pre-trained ResNet50 without classifier and pooling layer
        self.backbone = nn.Sequential(*list(torchvision.models.resnet50(pretrained=True, norm_layer=FrozenBatchNorm2d).children())[:-2])
        self.num_channels = 2048

    def forward(self, padded_batch, mask_batch):
        # extract the features from batch
        features = self.backbone(padded_batch)
        # Rescale masks to feature size
        feature_masks = F.interpolate(mask_batch.unsqueeze(0).float(), size=features.shape[-2:]).to(torch.bool)[0]
        return features, feature_masks
```


```python
backbone = Backbone()
padded_batch = padded_batch.float()
features, feature_masks = backbone(padded_batch, mask_batch)
print(f'Feature maps shape: {list(features.shape)} and feature masks shape: {list(feature_masks.shape)}')
```

    Feature maps shape: [2, 2048, 20, 20] and feature masks shape: [2, 20, 20]


### Add Positional Embeddings 
Positional embeddings are added to the feature maps to provide spatial information to the model. These embeddings help the transformer understand the positional relationships between different regions in the image.

The fixed positional embeddings are calculated using sine and cosine functions of different frequencies. These embeddings are then added to the input features. The choice of sine and cosine functions allows the model to capture periodic patterns in the positional information.

For a given position i and dimension d, the positional embedding is calculated as follows:
<div style="text-align:center">
    <img src='https://raw.githubusercontent.com/cenkbircanoglu/detr-blog/main/images/positional_embedding_sin.png' width=50% height=50%/>,
</div>
where D is the embedding dimension.

These fixed positional embeddings are then added to the input features before feeding them into the transformer model. This helps the model learn spatial relationships between different positions, even in scenarios where the input lacks explicit spatial information, such as images.


```python
# Initialise PositionEmbeddingSine class 
class PositionEmbeddingSine(nn.Module):
    def __init__(self, num_pos_feats=64, temperature=10000):
        super().__init__()
        self.num_pos_feats = num_pos_feats
        self.temperature = temperature
        self.scale = 2 * math.pi

    def forward(self, feature_masks):
        not_mask = ~feature_masks
        y_embed = not_mask.cumsum(1, dtype=torch.float32)
        x_embed = not_mask.cumsum(2, dtype=torch.float32)

        # Normalization of cumulative sums to ensure values are within a specified scale.
        eps = 1e-6
        y_embed = y_embed / (y_embed[:, -1:, :] + eps) * self.scale
        x_embed = x_embed / (x_embed[:, :, -1:] + eps) * self.scale

        # dim_t: A tensor representing the positional dimensions.
        dim_t = torch.arange(self.num_pos_feats, dtype=torch.float32)
        # Exponential scaling of dimensions based on the temperature hyperparameter.
        dim_t = self.temperature ** (2 * (dim_t // 2) / self.num_pos_feats)
        
        # Computation of positional embeddings for both x and y dimensions using sine and cosine functions.
        pos_x = x_embed[:, :, :, None] / dim_t
        pos_y = y_embed[:, :, :, None] / dim_t
        
        # Stack and flatten the sine and cosine components.
        pos_x = torch.stack((pos_x[:, :, :, 0::2].sin(), pos_x[:, :, :, 1::2].cos()), dim=4).flatten(3)
        pos_y = torch.stack((pos_y[:, :, :, 0::2].sin(), pos_y[:, :, :, 1::2].cos()), dim=4).flatten(3)
        # Concatenate x and y embeddings and permute dimensions to match the expected format.
        pos = torch.cat((pos_y, pos_x), dim=3).permute(0, 3, 1, 2)
        return pos
```


```python
# initialise position_embed module and generate embeddings from masks
embedding_dim = 256
N_steps = embedding_dim // 2

position_embed = PositionEmbeddingSine(N_steps)
positional_embeddings = position_embed(feature_masks).to(features.dtype)
print(f'Shape of the positional embeddings: {list(positional_embeddings.shape)}')
```

    Shape of the positional embeddings: [2, 256, 20, 20]


### High-level activation map

This operation is used to project the high-dimensional features into a lower-dimensional space, making it suitable for subsequent processing by the transformer layers. It allows the model to capture more abstract and compact representations of the input information.

<div style="text-align:center">
    <img src='https://raw.githubusercontent.com/cenkbircanoglu/detr-blog/main/images/high_level_activations.png' width=50% height=50%/>,
</div>



```python
# 1x1 convolution 
conv = nn.Conv2d(backbone.num_channels, embedding_dim, kernel_size=1)
activation_maps = conv(features)
print(f'High-level activation maps: {list(activation_maps.shape)} created from features: {list(features.shape)}')
```

    High-level activation maps: [2, 256, 20, 20] created from features: [2, 2048, 20, 20]


### Flatten Inputs
Since the transformer architecture doesn't accept images or patches as they are, the activation maps, positional embeddings, and feature masks are flattened as shown below. The spatial dimensions are collapsed into a single dimension, effectively transforming the 2D feature map into a sequence of tokens suitable for input to the transformer encoder.


```python
bs, c, h, w = activation_maps.shape
flattened_activation_maps = activation_maps.flatten(2).permute(2, 0, 1)
flattened_positional_embeddings = positional_embeddings.flatten(2).permute(2, 0, 1)
flattened_feature_masks = feature_masks.flatten(1)

print(f'Flattened and original activation maps shapes:{list(flattened_activation_maps.shape)}, {list(activation_maps.shape)} ')
print(f'Flattened and original positional embeddings shapes:{list(flattened_positional_embeddings.shape)}, {list(positional_embeddings.shape)} ')
print(f'Flattened and original feature masks shapes:{list(flattened_feature_masks.shape)}, {list(feature_masks.shape)} ')
```

    Flattened and original activation maps shapes:[400, 2, 256], [2, 256, 20, 20] 
    Flattened and original positional embeddings shapes:[400, 2, 256], [2, 256, 20, 20] 
    Flattened and original feature masks shapes:[2, 400], [2, 20, 20] 


### Multi-head attention

It aims to capture diverse relationships and dependencies within the input data by allowing the model to attend to different parts of the input sequence independently.


```python
# Initialise MultiHeadAttention class used in the original implementation
from torch.nn.modules.linear import NonDynamicallyQuantizableLinear


class MultiheadAttention(nn.Module):

    def __init__(self, embed_dim, num_heads, dropout=0.) -> None:
        super().__init__()
        self.embed_dim = embed_dim
        self.num_heads = num_heads
        self.dropout = dropout
        self.head_dim = embed_dim // num_heads

        self.in_proj_weight = nn.Parameter(torch.empty((3 * embed_dim, embed_dim)))
        self.in_proj_bias = nn.Parameter(torch.empty(3 * embed_dim))
        self.out_proj = NonDynamicallyQuantizableLinear(embed_dim, embed_dim, bias=True)

    def forward(self, query: torch.Tensor, key: torch.Tensor, value: torch.Tensor, key_padding_mask=None, attn_mask=None):

        # Allows the model to jointly attend to information from different representation subspaces as described 
        # in the paper: `Attention Is All You Need`.
        # Self-attention is being computed (i.e., ``query``, ``key``, and ``value`` are the same tensor).
        attn_output, attn_output_weights = F.multi_head_attention_forward(
            query, key, value, self.embed_dim, self.num_heads,
            self.in_proj_weight, self.in_proj_bias,
            None, None, False,
            self.dropout, self.out_proj.weight, self.out_proj.bias,
            key_padding_mask=key_padding_mask,
            attn_mask=attn_mask)
        return attn_output, attn_output_weights
```


```python
num_heads = 2
dropout = 0.1
self_attn = MultiheadAttention(embedding_dim, num_heads, dropout=dropout)

q = k = flattened_activation_maps + flattened_positional_embeddings
attn_output, attn_output_weights = self_attn(q, k, value=flattened_activation_maps, key_padding_mask=flattened_feature_masks)
print(f'Attention output shape: {list(attn_output.shape)} and Attention output weight shape: {list(attn_output_weights.shape)}')
```

    Attention output shape: [400, 2, 256] and Attention output weight shape: [2, 400, 400]


### Encoder Layer
The feature maps with positional embeddings are fed into a transformer encoder. The encoder processes the spatial information in parallel for different regions of the image. DETR uses a multi-head self-attention mechanism in the transformer encoder.


```python
# Initialise Single Layered Encoder module
class EncoderLayer(nn.Module):

    def __init__(self, embedding_dim, num_heads, hidden_dim=2048, dropout=0.1):
        super().__init__()
        self.self_attn = MultiheadAttention(embedding_dim, num_heads, dropout=dropout)
        # Implementation of Feedforward model
        self.linear1 = nn.Linear(embedding_dim, hidden_dim)
        self.dropout = nn.Dropout(dropout)
        self.linear2 = nn.Linear(hidden_dim, embedding_dim)

        self.norm1 = nn.LayerNorm(embedding_dim)
        self.norm2 = nn.LayerNorm(embedding_dim)
        self.dropout1 = nn.Dropout(dropout)
        self.dropout2 = nn.Dropout(dropout)

        self.activation = F.relu

    def forward(self, features, feature_mask, positional_embeddings):
        q = k = features + positional_embeddings
        src2 = self.self_attn(q, k, value=features, key_padding_mask=feature_mask)[0]
        features = features + self.dropout1(src2)
        features = self.norm1(features)
        src2 = self.linear2(self.dropout(self.activation(self.linear1(features))))
        features = features + self.dropout2(src2)
        features = self.norm2(features)
        return features
```


```python
dropout=0.1
hidden_dim=2048
encoder_layer = EncoderLayer(embedding_dim, num_heads, hidden_dim, dropout)

single_layer_encoder_output = encoder_layer(flattened_activation_maps, flattened_feature_masks, flattened_positional_embeddings)
print(f'Encoder output shape: {list(single_layer_encoder_output.shape)}')
```

    Encoder output shape: [400, 2, 256]


### Decoder Layer
The transformer decoder takes the output of the encoder and generates high-level feature representations. It utilizes multi-head self-attention and cross-attention mechanisms to capture both spatial and contextual information.


```python
# Initialise Single Layered Decoder module
class DecoderLayer(nn.Module):

    def __init__(self, embedding_dim, num_heads, hidden_dim=2048, dropout=0.1):
        super().__init__()
        self.self_attn = MultiheadAttention(embedding_dim, num_heads, dropout=dropout)
        self.multihead_attn = MultiheadAttention(embedding_dim, num_heads, dropout=dropout)
        # Implementation of Feedforward model
        self.linear1 = nn.Linear(embedding_dim, hidden_dim)
        self.dropout = nn.Dropout(dropout)
        self.linear2 = nn.Linear(hidden_dim, embedding_dim)

        self.norm1 = nn.LayerNorm(embedding_dim)
        self.norm2 = nn.LayerNorm(embedding_dim)
        self.norm3 = nn.LayerNorm(embedding_dim)
        self.dropout1 = nn.Dropout(dropout)
        self.dropout2 = nn.Dropout(dropout)
        self.dropout3 = nn.Dropout(dropout)

        self.activation = F.relu

    def forward(self, target, encoder_outputs, feature_masks, positional_embeddings, query_embeddings):
        q = k = target + query_embeddings
        tgt2 = self.self_attn(q, k, value=target)[0]
        target = target + self.dropout1(tgt2)
        target = self.norm1(target)
        tgt2 = self.multihead_attn(query=target + query_embeddings,
                                   key=encoder_outputs + positional_embeddings,
                                   value=encoder_outputs, key_padding_mask=feature_masks)[0]
        target = target + self.dropout2(tgt2)
        target = self.norm2(target)
        tgt2 = self.linear2(self.dropout(self.activation(self.linear1(target))))
        target = target + self.dropout3(tgt2)
        target = self.norm3(target)
        return target
```

#### Object Queries
Object queries are associated with specific object classes that the model is trained to recognize. The attention mechanism guided by these queries helps the model determine the presence of objects and predict their corresponding class labels.
The attended features from different object queries are used collectively to generate predictions for both object localization (bounding boxes) and object classification (class labels). The model leverages the information gathered by object queries to make accurate predictions


```python
decoder_layer = DecoderLayer(embedding_dim, num_heads, hidden_dim, dropout)

num_queries = 100
query_embeddings = nn.Embedding(num_queries, embedding_dim).weight.unsqueeze(1).repeat(1, bs, 1)

target = torch.zeros_like(query_embeddings)
single_layer_decoder_output = decoder_layer(target, single_layer_encoder_output, flattened_feature_masks, flattened_positional_embeddings, query_embeddings)
print(f'Decoder output shape: {list(single_layer_decoder_output.shape)}')
```

    Decoder output shape: [100, 2, 256]


### Multi-Layer Encoder Model
The feature maps with positional embeddings are fed into a transformer encoder. The encoder processes the spatial information in parallel for different regions of the image. DETR uses a multi-head self-attention mechanism in the transformer encoder.


```python
# Multi-layer encoder model which only creates N encoder layer and feeds each other in the execution step.
class TransformerEncoder(nn.Module):

    def __init__(self, embedding_dim, num_heads, hidden_dim, dropout, num_layers):
        super().__init__()
        encoder_layer = EncoderLayer(embedding_dim, num_heads, hidden_dim, dropout)
        self.layers = nn.ModuleList([encoder_layer for _ in range(num_layers)])

    def forward(self, features, feature_mask, positional_embeddings):
        output = features

        for layer in self.layers:
            output = layer(output, feature_mask, positional_embeddings)

        return output
```


```python
num_encoder_layers=6
encoder = TransformerEncoder(embedding_dim, num_heads, hidden_dim, dropout, num_encoder_layers)

encoder_output = encoder(flattened_activation_maps, flattened_feature_masks, flattened_positional_embeddings)
assert encoder_output.shape == single_layer_encoder_output.shape
print(f'Encoder output shape: {list(encoder_output.shape)}')
```

    Encoder output shape: [400, 2, 256]


### Multi head Decoder Model


```python
# Multi-layer decoder model which only creates N decoder layer and feeds each other in the execution step.
class TransformerDecoder(nn.Module):

    def __init__(self, embedding_dim, num_heads, hidden_dim, dropout, num_layers):
        super().__init__()
        decoder_layer = DecoderLayer(embedding_dim, num_heads, hidden_dim, dropout)

        self.layers = nn.ModuleList([decoder_layer for _ in range(num_layers)])
        self.norm = nn.LayerNorm(embedding_dim)

    def forward(self, target, encoder_outputs, feature_masks, positional_embeddings, query_embeddings):
        output = target

        for layer in self.layers:
            output = layer(output, encoder_outputs, feature_masks,positional_embeddings, query_embeddings)

        if self.norm is not None:
            output = self.norm(output)

        return output.unsqueeze(0)
```


```python
num_decoder_layers=6
decoder = TransformerDecoder(embedding_dim, num_heads, hidden_dim, dropout, num_encoder_layers)
decoder_output = decoder(target, encoder_output, flattened_feature_masks, flattened_positional_embeddings, query_embeddings)
assert decoder_output[0].shape == single_layer_decoder_output.shape 
print(f'Decoder output shape: {list(decoder_output.shape)}')
```

    Decoder output shape: [1, 100, 2, 256]


### Transformer Model
The transformer model combines both encoder and decoder components in its architecture and performs the flattening operations within its forward function. The forward function is a crucial part of the model where input data is processed, and predictions are generated. 


```python
# Introduce Transformer model
class Transformer(nn.Module):

    def __init__(self, embedding_dim=512, num_heads=8, hidden_dim=2048, dropout=0.1, num_encoder_layers=6, num_decoder_layers=6):
        super().__init__()

        self.encoder = TransformerEncoder(embedding_dim, num_heads, hidden_dim, dropout, num_encoder_layers)
        self.decoder = TransformerDecoder(embedding_dim, num_heads, hidden_dim, dropout, num_decoder_layers)

    def forward(self, features, feature_masks, query_embeddings, positional_embeddings):
        bs, c, h, w = features.shape
        features = features.flatten(2).permute(2, 0, 1)
        positional_embeddings = positional_embeddings.flatten(2).permute(2, 0, 1)
        query_embeddings = query_embeddings.unsqueeze(1).repeat(1, bs, 1)
        feature_masks = feature_masks.flatten(1)

        target = torch.zeros_like(query_embeddings)
        encoder_output = self.encoder(features, feature_masks, positional_embeddings)
        decoder_output = self.decoder(target, encoder_output, feature_masks, positional_embeddings, query_embeddings)
        return decoder_output.transpose(1, 2), encoder_output.permute(1, 2, 0).view(bs, c, h, w)
```


```python
# initialise transformer model
num_decoder_layers = 6
num_encoder_layers = 6
dropout = 0.1
num_heads = 8
hidden_dim = 2048
embedding_dim = 256

transformer = Transformer(embedding_dim=embedding_dim, dropout=dropout, num_heads=num_heads, hidden_dim=hidden_dim, 
                          num_encoder_layers=num_encoder_layers, num_decoder_layers=num_decoder_layers)

# initialise Query Embedding
num_queries = 100
query_embeddings = nn.Embedding(num_queries, embedding_dim).weight

# execute transformer model
decoder_output, encoder_output = transformer(activation_maps, feature_masks, query_embeddings, positional_embeddings)
print(f'Shape of decoder output: {list(decoder_output.shape)} and encoder output: {list(encoder_output.shape)}')
```

    Shape of decoder output: [1, 2, 100, 256] and encoder output: [2, 256, 20, 20]


### Classification head
A linear layer is used for predicting the class probabilities for each bounding box. It takes the decoder's high-level features as input and outputs class scores.


```python
num_classes = 20
class_embed = nn.Linear(embedding_dim, num_classes + 1)
classifier_output = class_embed(decoder_output)
print(f'Classifier outputs: {list(classifier_output.shape)}')
```

    Classifier outputs: [1, 2, 100, 21]


### MLP Model
The MLP (Multi-Layer Perceptron) head is responsible for predicting bounding box coordinates. It takes the high-level features from the decoder and produces bounding box coordinates for each object in the image.


```python
class MLP(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, num_layers):
        super().__init__()
        self.num_layers = num_layers
        h = [hidden_dim] * (num_layers - 1)
        self.layers = nn.ModuleList(nn.Linear(n, k) for n, k in zip([input_dim] + h, h + [output_dim]))

    def forward(self, x):
        for i, layer in enumerate(self.layers):
            x = F.relu(layer(x)) if i < self.num_layers - 1 else layer(x)
        return x
```


```python
bbox_embed = MLP(embedding_dim, embedding_dim, 4, 3)
mlp_output = bbox_embed(decoder_output).sigmoid()
print(f'MLP outputs: {list(mlp_output.shape)}')
```

    MLP outputs: [1, 2, 100, 4]


### DETR Model

At the end, we can create the DETR model by combining the models defined above according to the what authors described.


```python
class DETR(nn.Module):
    def __init__(self, embedding_dim=2048, dropout=0.1, num_heads=8, hidden_dim=2048, num_encoder_layers=6, 
                 num_decoder_layers=6, num_classes=20, num_queries=100):
        super().__init__()

        # Initialiase backbone
        self.backbone = Backbone()
        
        # Initialise PositionEmbeddings
        N_steps = embedding_dim // 2
        self.position_embedding = PositionEmbeddingSine(N_steps)
        
        # Initialise Transformer Model
        self.transformer = Transformer(embedding_dim=embedding_dim, dropout=dropout, num_heads=num_heads, hidden_dim=hidden_dim, 
                                       num_encoder_layers=num_encoder_layers, num_decoder_layers=num_decoder_layers)
        
        # initialise classifier
        self.classifier = nn.Linear(embedding_dim, num_classes + 1)

        # initialise bounding box model
        self.bbox_mlp = MLP(embedding_dim, embedding_dim, 4, 3)

        # initialise query embeddings
        self.query_embeddings = nn.Embedding(num_queries, embedding_dim)
        self.input_proj = nn.Conv2d(backbone.num_channels, embedding_dim, kernel_size=1)
        

    def forward(self, batch):
        # Resize the batch and create masks
        padded_batch, mask_batch = resize_images_and_create_masks(batch)
        padded_batch = padded_batch.float()
        
        # Extract features from backbone
        features, feature_masks = self.backbone(padded_batch, mask_batch)

        # initialise positional embeddings
        positional_embeddings = self.position_embedding(feature_masks).to(features.dtype)

        # execute transformer model
        decoder_output, encoder_output = self.transformer(self.input_proj(features), feature_masks, self.query_embeddings.weight, positional_embeddings)

        # classify decoder outputs
        classifier_output = self.classifier(decoder_output)
        
        # execute bounding box mlp model
        mlp_output = self.bbox_mlp(decoder_output).sigmoid()
        
        out = {'pred_logits': classifier_output[-1], 'pred_boxes': mlp_output[-1]}
        return out
```


```python
# Initialise the model
model = DETR(embedding_dim=2048, dropout=0.1, num_heads=8, hidden_dim=2048, num_encoder_layers=6, num_decoder_layers=6, 
        num_classes=20, num_queries=100)

outputs = model(image_list)
print(f"Predicted logits and predicted boxes have shapes: {list(outputs['pred_logits'].shape)}, {list(outputs['pred_boxes'].shape)}")
```

    Max Width and Height value in the batch: 640, 639
