# Action Chunking with Transformers

## Training

![Architecture of Action Chunking with Transformers (ACT)](assets/image-20230816130125739.png)

The whole model is trained as a Conditional VAE, which means it first produces a latent code extracted from the input (called encoder, left in the picture), and then uses the latent code to restore the input (called decoder, right in the picture). In details, it regards observations as the conditions to constrain and help itself to perform better.

Before training, we should to create our `dataloader`. First, its `.dhf5` file data structure is like this below:

```tex
action: target joint positions [episode_len, 14] # each robot has 7 DoF
observations:
	- images: [episode_len, cam_num, h, w]
		- top
		- left wrist
		- right wrist
		- front
	- qpos: current joint positions [episode_len, 14]
	- qvel: current joint velocity [episode_len, 14]
```

**Sample the data:** Here it just randomly select an int from [0, episode_len], named `start_ts`. For per data sample in a batch, we just select index of `start_ts` from original data. Specifically, `dataset[observations][qpos][start_ts]` for instance, and the images as well. About action sequence, we do the follow things.

```python
# create a zero vector in action episode shape
padded_action = torch.zero_like(action)
# sample actions from action episode
sample_actions = dataset[action][start_ts:]
# replace padded_action with sample actions
padded_action[:len(sample_actions)] = sample_actions
# sample the first k actions as action sequence
action_sequence = padded_action[:k]
```

### Input

```python
# joints: joint position/torques
# action sequence: a sequence of action in order
# images: the images store image info in every single timestep from wrist, front, top cameras 
```

### CVAE Encoder

![image-20230816143139064](assets/image-20230816143139064.png)

```python
## Obtain latent z from action sequence
# project action sequence to embedding dim, and concat with a CLS token
action_embed = self.encoder_action_proj(actions) # (bs, seq, hidden_dim)
qpos_embed = self.encoder_joint_proj(qpos)  # (bs, hidden_dim)
qpos_embed = torch.unsqueeze(qpos_embed, axis=1)  # (bs, 1, hidden_dim)
cls_embed = self.cls_embed.weight # (1, hidden_dim)
cls_embed = torch.unsqueeze(cls_embed, axis=0).repeat(bs, 1, 1) # (bs, 1, hidden_dim)
encoder_input = torch.cat([cls_embed, qpos_embed, action_embed], axis=1) # (bs, seq+2, hidden_dim)
encoder_input = encoder_input.permute(1, 0, 2) # (seq+2, bs, hidden_dim)
# do not mask cls token
cls_joint_is_pad = torch.full((bs, 2), False).to(qpos.device) # False: not a padding
is_pad = torch.cat([cls_joint_is_pad, is_pad], axis=1)  # (bs, seq+1)
# obtain position embedding
pos_embed = self.pos_table.clone().detach()
pos_embed = pos_embed.permute(1, 0, 2)  # (seq+1, 1, hidden_dim)
# query model
encoder_output = self.encoder(encoder_input, pos=pos_embed, src_key_padding_mask=is_pad)
encoder_output = encoder_output[0] # take cls output only
latent_info = self.latent_proj(encoder_output)
mu = latent_info[:, :self.latent_dim]
logvar = latent_info[:, self.latent_dim:]
latent_sample = reparametrize(mu, logvar)
latent_input = self.latent_out_proj(latent_sample)
```

> It will concatenate embedded CLS (bs, 1, dim), embedded joint positions (bs, 1 , dim), and action sequence (bs, horizon, dim) which has time history information.
>
> Here, in each transformer encoder layer, it will do one time position embedding, which is a big difference from pytorch package. (Here position embedding is not learnable)

### Visual Encoder

<div align='center'>
    <img src='assets/visual_encoder.png' width="400" />
</div>

```python
class FrozenBatchNorm2d(torch.nn.Module):
    """
    BatchNorm2d where the batch statistics and the affine parameters are fixed.

    Copy-paste from torchvision.misc.ops with added eps before rqsrt,
    without which any other policy_models than torchvision.policy_models.resnet[18,34,50,101]
    produce nans.
    """

    def __init__(self, n):
        super(FrozenBatchNorm2d, self).__init__()
        self.register_buffer("weight", torch.ones(n))
        self.register_buffer("bias", torch.zeros(n))
        self.register_buffer("running_mean", torch.zeros(n))
        self.register_buffer("running_var", torch.ones(n))

    def _load_from_state_dict(self, state_dict, prefix, local_metadata, strict,
                              missing_keys, unexpected_keys, error_msgs):
        num_batches_tracked_key = prefix + 'num_batches_tracked'
        if num_batches_tracked_key in state_dict:
            del state_dict[num_batches_tracked_key]

        super(FrozenBatchNorm2d, self)._load_from_state_dict(
            state_dict, prefix, local_metadata, strict,
            missing_keys, unexpected_keys, error_msgs)

    def forward(self, x):
        # move reshapes to the beginning
        # to make it fuser-friendly
        w = self.weight.reshape(1, -1, 1, 1)
        b = self.bias.reshape(1, -1, 1, 1)
        rv = self.running_var.reshape(1, -1, 1, 1)
        rm = self.running_mean.reshape(1, -1, 1, 1)
        eps = 1e-5
        scale = w * (rv + eps).rsqrt()
        bias = b - rm * scale
        return x * scale + bias


class BackboneBase(nn.Module):

    def __init__(self, backbone: nn.Module, train_backbone: bool, num_channels: int, return_interm_layers: bool):
        super().__init__()
	return_layers = {'layer4': "0"}
        self.body = IntermediateLayerGetter(backbone, return_layers=return_layers)
        self.num_channels = num_channels

    def forward(self, tensor):
        xs = self.body(tensor)
        return xs
        # out: Dict[str, NestedTensor] = {}
        # for name, x in xs.items():
        #     m = tensor_list.mask
        #     assert m is not None
        #     mask = F.interpolate(m[None].float(), size=x.shape[-2:]).to(torch.bool)[0]
        #     out[name] = NestedTensor(x, mask)
        # return out


class Backbone(BackboneBase):
    def __init__(self, name: str,
                 train_backbone: bool,
                 return_interm_layers: bool,
                 dilation: bool):
        backbone = getattr(torchvision.models, name)(
            replace_stride_with_dilation=[False, False, dilation],
            pretrained=is_main_process(), norm_layer=FrozenBatchNorm2d) # change nn.BatchNorm2d to FrozenBatchNorm2d
        num_channels = 512 if name in ('resnet18', 'resnet34') else 2048
        super().__init__(backbone, train_backbone, num_channels, return_interm_layers)

class Joiner(nn.Sequential):
    def __init__(self, 
                 backbone: Backbone, 
                 position_embedding: Union[PositionEmbeddingLearned, PositionEmbeddingSine]):
        super().__init__(backbone, position_embedding)

    def forward(self, tensor_list: NestedTensor):
        xs = self[0](tensor_list)
        out: List[NestedTensor] = []
        pos = []
        for name, x in xs.items():
            out.append(x)
            # position encoding
            pos.append(self[1](x).to(x.dtype))

        return out, pos
```

### CVAE Decoder

<div align='center'>
    <img src='assets/cvae_decoder.png' />
</div>

> As the code describes below, we feed `src, query, img_pos, latent_input, joints (proprio_input), additional_pos` into CVAE Encoder.

```python
"""
qpos: batch, qpos_dim
image: batch, num_cam, channel, height, width
env_state: None
actions: batch, seq, action_dim
"""
# ......
self.input_proj = nn.Conv2d(backbones[0].num_channels, hidden_dim, kernel_size=1)
self.backbones = nn.ModuleList(backbones)
self.input_proj_robot_state = nn.Linear(14, hidden_dim)

# Image observation features and position embeddings
all_cam_features = []
all_cam_pos = []
for cam_id, cam_name in enumerate(self.camera_names):
    features, pos = self.backbones[0](image[:, cam_id]) # HARDCODED
    features = features[0] # take the last layer feature
    pos = pos[0]
    all_cam_features.append(self.input_proj(features))
    all_cam_pos.append(pos)
# proprioception features
proprio_input = self.input_proj_robot_state(qpos)
# fold camera dimension into width dimension
src = torch.cat(all_cam_features, axis=3)
pos = torch.cat(all_cam_pos, axis=3)
hs = self.transformer(src, None, self.query_embed.weight, pos, latent_input, proprio_input, self.additional_pos_embed.weight)[0]
a_hat = self.action_head(hs)
is_pad_hat = self.is_pad_head(hs)
```

Then inside CVAE Encoder, it will do follow things to fit transformer inputs. More specifically, transformer encoder in CVAE Decoder, it takes in latent_input, joints and image features. Meanwhile, the transformer decoder takes in tgt, output from transformer encoder.

```python
# flatten NxCxHxW to HWxNxC
bs, c, h, w = src.shape
src = src.flatten(2).permute(2, 0, 1)
pos_embed = pos_embed.flatten(2).permute(2, 0, 1).repeat(1, bs, 1)
query_embed = query_embed.unsqueeze(1).repeat(1, bs, 1)
# mask = mask.flatten(1)

additional_pos_embed = additional_pos_embed.unsqueeze(1).repeat(1, bs, 1) # seq, bs, dim
pos_embed = torch.cat([additional_pos_embed, pos_embed], axis=0)

addition_input = torch.stack([latent_input, proprio_input], axis=0)
src = torch.cat([addition_input, src], axis=0)

tgt = torch.zeros_like(query_embed)
memory = self.encoder(src, src_key_padding_mask=mask, pos=pos_embed)
hs = self.decoder(tgt, memory, memory_key_padding_mask=mask,
                  pos=pos_embed, query_pos=query_embed)
```



> It selects the output of {'layer4': "0"} (transfer shape: 3, h, w into shape: dim, h’, w’) to represent images features, whose shape is (bs. dim, cam_num * h’ * w’). And its position information is based on the relative position of (h’, w’). (not learnable)
>
> Secondly, it changes a part of resnet18, which uses FrozenBatchNorm2d to replace all NormLayers.

```python
# transformer decoder forward
q = k = self.with_pos_embed(tgt, query_pos)
tgt2 = self.self_attn(q, k, value=tgt, attn_mask=tgt_mask,
                      key_padding_mask=tgt_key_padding_mask)[0]
tgt = tgt + self.dropout1(tgt2)
tgt = self.norm1(tgt)
tgt2 = self.multihead_attn(query=self.with_pos_embed(tgt, query_pos),
                           key=self.with_pos_embed(memory, pos),
                           value=memory, attn_mask=memory_mask,
                           key_padding_mask=memory_key_padding_mask)[0]
```



> **Encoder**
>
> In transformer encoder, it concatenates images features (bs, cam_num * h * 2, dim), joints positions (bs, 1, dim) and latent code (bs, 1, dim). And do position embedding in each encoder layer. (Here one part of position embedding is from Visual Encoder, another is nn.Embedding.weights).
>
> **Decoder**
>
> The input (namely queries) are learnable parameters. In every decoder layer, it would not just embed during self-attention, but also embed in multi-head attention.

### Loss Function

The total loss contains l1_loss/MAE and kl divergence. l1_loss is got by calculating the difference between predict_actions and original actions. kl divergence is calculated with mean and variance.

```python
def __call__(self, qpos, image, actions=None, is_pad=None):
    env_state = None
    normalize = transforms.Normalize(mean=[0.485, 0.456, 0.406],
                                     std=[0.229, 0.224, 0.225])
    image = normalize(image)
    if actions is not None: # training time
        actions = actions[:, :self.model.num_queries]
        is_pad = is_pad[:, :self.model.num_queries]

        a_hat, is_pad_hat, (mu, logvar) = self.model(qpos, image, env_state, actions, is_pad)
        total_kld, dim_wise_kld, mean_kld = kl_divergence(mu, logvar)
        loss_dict = dict()
        all_l1 = F.l1_loss(actions, a_hat, reduction='none')
        l1 = (all_l1 * ~is_pad.unsqueeze(-1)).mean()
        loss_dict['l1'] = l1
        loss_dict['kl'] = total_kld[0]
        loss_dict['loss'] = loss_dict['l1'] + loss_dict['kl'] * self.kl_weight
        return loss_dict
    else: # inference time
        import ipdb
        # ipdb.set_trace()
        a_hat, _, (_, _) = self.model(qpos, image, env_state) # no action, sample from prior
        return a_hat
```



## Inference

When doing testing, `Encoder` will be discarded and z is simply set to the mean of the prior (i.e. zero) at test time.

![image-20230816143339714](assets/image-20230816143339714.png)

```python
""" set latent input """
mu = logvar = None
latent_sample = torch.zeros([bs, self.latent_dim], dtype=torch.float32).to(qpos.device)
latent_input = self.latent_out_proj(latent_sample)
```

In testing time, policy will return actions instead of loss_dict (can be seen in Loss Function section).

## Code Structure

Code structure here is pretty easy, but when you need to modify something, it becomes not so convenient.

```bash
.
├── assets
├── data_dir
├── detr
├── constants.py
├── utils.py
├── ee_sim_env.py
├── sim_env.py
├── policy.py
├── scripted_policy.py
├── imitate_episodes.py
├── record_sim_episodes.py
└── visualize_episodes.py
```

`assets` saves the robot physical modle(xml file) and various environments in different tasks.
`data_dir` saves traning data.
`detr` is the transformer model file, defines the model structure.
`utils` defines some common used functions.
`ee_sim_env/sim_env` respectively defines EEF-control and Joint-Pos control space in Mujoco Env.
`scripted_policy` is a hand-made policy to lead the robot how to complete the task.
`policy` is a policy used detr model.
`imitate_episodes` defines how it train and evaluate the whole model.
`record_sim_episodes` is to produce simulation data.
`visualize_episodes` is a method to save the simulation video.  
