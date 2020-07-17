# Anatomy of Detection Transformer(DETR)

## COCO
```
IoU metric: bbox
 Average Precision  (AP) @[ IoU=0.50:0.95 | area=   all | maxDets=100 ] = 0.416
 Average Precision  (AP) @[ IoU=0.50      | area=   all | maxDets=100 ] = 0.616
 Average Precision  (AP) @[ IoU=0.75      | area=   all | maxDets=100 ] = 0.442
 Average Precision  (AP) @[ IoU=0.50:0.95 | area= small | maxDets=100 ] = 0.206
 Average Precision  (AP) @[ IoU=0.50:0.95 | area=medium | maxDets=100 ] = 0.447
 Average Precision  (AP) @[ IoU=0.50:0.95 | area= large | maxDets=100 ] = 0.609
 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets=  1 ] = 0.334
 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets= 10 ] = 0.535
 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets=100 ] = 0.577
 Average Recall     (AR) @[ IoU=0.50:0.95 | area= small | maxDets=100 ] = 0.311
 Average Recall     (AR) @[ IoU=0.50:0.95 | area=medium | maxDets=100 ] = 0.629
 Average Recall     (AR) @[ IoU=0.50:0.95 | area= large | maxDets=100 ] = 0.814
Training time 12 days, 16:01:22
```

## Abstract

- DETR has three main parts, which are transformer, backbone and matcher.
- First of all, a DETR matcher is for matching between groundtruth and predictions with Hungarian Algorithm, also known as the Munkres or Kuhn-Munkres algorithm.
- Secondly, a DETR has a transformer which contains Multi-head-attention as main modules.
- Finaly, DETR use resnet-50(or resnet-101) network as a image feature extractor.
- Additionaly, to compute losses, DETR use Hungarian-matched permutation and GIOU for labels loss and boxes loss.
- DETR also some embeddings and encodings for query and postion handling in a Transformer.

## Backbone feature extractor
- Input : scale augmented from shortest 480(to 800) to longest 1333. 3 x H x W
- Output : 2048 x H / 32 x W / 32
- for example : feature resultion by layers = 640-320-160-80-40-20
- final image feature resulution is 20 x 20 (for 640 x 640 image)

## Hungarian Matcher

```
        # models/matcher.py
        # We flatten to compute the cost matrices in a batch
        out_prob = outputs["pred_logits"].flatten(0, 1).softmax(-1)  # [batch_size * num_queries, num_classes]
        out_bbox = outputs["pred_boxes"].flatten(0, 1)  # [batch_size * num_queries, 4]

        # Also concat the target labels and boxes
        tgt_ids = torch.cat([v["labels"] for v in targets])
        tgt_bbox = torch.cat([v["boxes"] for v in targets])

        # Compute the classification cost. Contrary to the loss, we don't use the NLL,
        # but approximate it in 1 - proba[target class].
        # The 1 is a constant that doesn't change the matching, it can be ommitted.
        cost_class = -out_prob[:, tgt_ids]

        # Compute the L1 cost between boxes
        cost_bbox = torch.cdist(out_bbox, tgt_bbox, p=1)

        # Compute the giou cost betwen boxes
        cost_giou = -generalized_box_iou(box_cxcywh_to_xyxy(out_bbox), box_cxcywh_to_xyxy(tgt_bbox))
# Final cost matrix
        C = self.cost_bbox * cost_bbox + self.cost_class * cost_class + self.cost_giou * cost_giou
        C = C.view(bs, num_queries, -1).cpu()

        sizes = [len(v["boxes"]) for v in targets]
        indices = [linear_sum_assignment(c[i]) for i, c in enumerate(C.split(sizes, -1))]
```
## Transformer
- d_model(=hidden_dims) : 512
- nhead : 8
- num_encoder_layers : 6
- num_decoder_layers : 6
- dim_feedforward : 2048

```
# models/transformer.py
    def forward(self, src, mask, query_embed, pos_embed):
        # flatten NxCxHxW to HWxNxC
        bs, c, h, w = src.shape
        src = src.flatten(2).permute(2, 0, 1)
        pos_embed = pos_embed.flatten(2).permute(2, 0, 1)
        query_embed = query_embed.unsqueeze(1).repeat(1, bs, 1)
        mask = mask.flatten(1)

        tgt = torch.zeros_like(query_embed)
        memory = self.encoder(src, src_key_padding_mask=mask, pos=pos_embed)
        hs = self.decoder(tgt, memory, memory_key_padding_mask=mask,
                          pos=pos_embed, query_pos=query_embed)
        return hs.transpose(1, 2), memory.permute(1, 2, 0).view(bs, c, h, w)
```        

## Transformer Input/Output Representation and Handling

### Spatial Positional Encoding/Embedding
- Along the 2-d axis X and Y each, allocate sine values for even indices and cosine values for odd indices.
```
e.g. [X, Y]
[[sin0, sin0], [cos0.25, sin0], [sin0.5, sin0], [sin0.75, sin0], [sin1.0, sin0],
[sin0, cos0.25], [cos0.25, cos0.25], [sin0.5, cos0.25], [sin0.75, cos0.25], [sin1.0, cos0.25],
[sin0, sin0.5], [cos0.25, sin0.5], [sin0.5, sin0.5], [sin0.75, sin0.5], [sin1.0, sin0.5],
[sin0, cos0.75], [cos0.25, cos0.75], [sin0.5, cos0.75], [sin0.75, cos0.75], [sin1.0, cos0.75],
[sin0, sin1.0], [cos0.25, sin1.0], [sin0.5, sin1.0], [sin0.75, sin1.0], [sin1.0, sin1.0]]
```
```
class PositionEmbeddingSine(nn.Module):
    """
    This is a more standard version of the position embedding, very similar to the one
    used by the Attention is all you need paper, generalized to work on images.
    """
...
    def forward(self, tensor_list: NestedTensor):
        x = tensor_list.tensors
        mask = tensor_list.mask
        assert mask is not None
        not_mask = ~mask
        y_embed = not_mask.cumsum(1, dtype=torch.float32)
        x_embed = not_mask.cumsum(2, dtype=torch.float32)
        if self.normalize:
            eps = 1e-6
            y_embed = y_embed / (y_embed[:, -1:, :] + eps) * self.scale
            x_embed = x_embed / (x_embed[:, :, -1:] + eps) * self.scale

        dim_t = torch.arange(self.num_pos_feats, dtype=torch.float32, device=x.device)
        dim_t = self.temperature ** (2 * (dim_t // 2) / self.num_pos_feats)

        pos_x = x_embed[:, :, :, None] / dim_t
        pos_y = y_embed[:, :, :, None] / dim_t
        pos_x = torch.stack((pos_x[:, :, :, 0::2].sin(), pos_x[:, :, :, 1::2].cos()), dim=4).flatten(3)
        pos_y = torch.stack((pos_y[:, :, :, 0::2].sin(), pos_y[:, :, :, 1::2].cos()), dim=4).flatten(3)
        pos = torch.cat((pos_y, pos_x), dim=3).permute(0, 3, 1, 2)
        return pos
```        
- pos in
```
# models/detr.py
class DETR(nn.Module):
    """ This is the DETR module that performs object detection """
...
    def forward(self, samples: NestedTensor):
...    
        features, pos = self.backbone(samples)
...

        src, mask = features[-1].decompose()
        assert mask is not None
        hs = self.transformer(self.input_proj(src), mask, self.query_embed.weight, pos[-1])[0]
```
```
# models/transformer.py
class TransformerEncoderLayer(nn.Module):
...
    def with_pos_embed(self, tensor, pos: Optional[Tensor]):
        return tensor if pos is None else tensor + pos
...
    def forward_post(self,
                     src,
                     src_mask: Optional[Tensor] = None,
                     src_key_padding_mask: Optional[Tensor] = None,
                     pos: Optional[Tensor] = None):
        q = k = self.with_pos_embed(src, pos)
        src2 = self.self_attn(q, k, value=src, attn_mask=src_mask,
                              key_padding_mask=src_key_padding_mask)[0]
```
- pos_embed in
```
# models/transformer.py
class Transformer(nn.Module):
...
    def forward(self, src, mask, query_embed, pos_embed):
        # flatten NxCxHxW to HWxNxC
        bs, c, h, w = src.shape
        src = src.flatten(2).permute(2, 0, 1)
        pos_embed = pos_embed.flatten(2).permute(2, 0, 1)
        query_embed = query_embed.unsqueeze(1).repeat(1, bs, 1)
        mask = mask.flatten(1)

        tgt = torch.zeros_like(query_embed)
        memory = self.encoder(src, src_key_padding_mask=mask, pos=pos_embed)
        hs = self.decoder(tgt, memory, memory_key_padding_mask=mask,
                          pos=pos_embed, query_pos=query_embed)
        return hs.transpose(1, 2), memory.permute(1, 2, 0).view(bs, c, h, w)
```

### Query Embedding
- = Object Queries
- = Output Positional Encoding
- query_pos in
```
# models/transformer.py
class TransformerDecoder(nn.Module):
....
    def forward(self, tgt, memory,
                tgt_mask: Optional[Tensor] = None,
                memory_mask: Optional[Tensor] = None,
                tgt_key_padding_mask: Optional[Tensor] = None,
                memory_key_padding_mask: Optional[Tensor] = None,
                pos: Optional[Tensor] = None,
                query_pos: Optional[Tensor] = None):
```
- query_embed in 
```
# models/transformer.py
class Transformer(nn.Module):
...
    def forward(self, src, mask, query_embed, pos_embed):
        # flatten NxCxHxW to HWxNxC
        bs, c, h, w = src.shape
        src = src.flatten(2).permute(2, 0, 1)
        pos_embed = pos_embed.flatten(2).permute(2, 0, 1)
        query_embed = query_embed.unsqueeze(1).repeat(1, bs, 1)
        mask = mask.flatten(1)

        tgt = torch.zeros_like(query_embed)
        memory = self.encoder(src, src_key_padding_mask=mask, pos=pos_embed)
        hs = self.decoder(tgt, memory, memory_key_padding_mask=mask,
                          pos=pos_embed, query_pos=query_embed)
        return hs.transpose(1, 2), memory.permute(1, 2, 0).view(bs, c, h, w)
```
```
# models/detr.py
class DETR(nn.Module):
    """ This is the DETR module that performs object detection """
    def __init__(self, backbone, transformer, num_classes, num_queries, aux_loss=False):
    ...
            self.query_embed = nn.Embedding(num_queries, hidden_dim)
    ...
    def forward(self, samples: NestedTensor):
    ...
    hs = self.transformer(self.input_proj(src), mask, self.query_embed.weight, pos[-1])[0]
    ...    
```
## GIOU

## Input Data Prepocessing
### Scale Augmentation

## Comparison to the single step object detectors
- no feature pyramid
- no SPP
- Resnet 50 basic
- no key points
