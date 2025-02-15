# text_recognizer/models/line_cnn_transformer.py

```python
"""
How to make inference faster: https://pgresia.medium.com/making-pytorch-transformer-twice-as-fast-on-sequence-generation-2a8a7f1e7389
"""
import argparse
from typing import Any, Dict
import math
import torch
import torch.nn

from .line_cnn import LineCNN


TF_DIM = 256
TF_FC_DIM = 1024
TF_DROPOUT = 0.4
TF_LAYERS = 4
TF_NHEAD = 4


class PositionalEncoding(torch.nn.Module):
    """Classic Attention-is-all-you-need positional encoding."""

    def __init__(self, d_model: int, dropout: float = 0.1, max_len: int = 5000) -> None:
        super(PositionalEncoding, self).__init__()
        self.dropout = torch.nn.Dropout(p=dropout)

        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len, dtype=torch.float).unsqueeze(1) # 상자 씌우기, shape=(max_len,) -> (max_len,1)
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model)) # exp(- 2k * log10000/ d) = (10^4)^(-2k/d)
        pe[:, 0::2] = torch.sin(position * div_term) # phase = position * div_term , phase.shape = (max_len, d_model/2)
        pe[:, 1::2] = torch.cos(position * div_term)
        pe = pe.unsqueeze(0).transpose(0, 1) # pe.shape = (1, max_len, d_model) -> (max_len, 1, d_model)
        self.register_buffer("pe", pe)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = x + self.pe[: x.size(0), :] # if pe.shape = (10, 1, 8) and x.size(0) =3, self.pe[: x.size(0), :].shape = (3, 1, 8)
        return self.dropout(x)


def generate_square_subsequent_mask(size: int) -> torch.Tensor:
    """Generate a triangular (size, size) mask."""
    mask = (torch.triu(torch.ones(size, size)) == 1).transpose(0, 1) # upper True matrix -> transpose. therefore lower triangle
    mask = mask.float().masked_fill(mask == 0, float("-inf")).masked_fill(mask == 1, float(0.0))
    return mask
```
positional encoding을 max_length에 대해 수행한 결과를 untrainable layer로 저장합니다. `self.register_buffer('pe', pe)`로 생성된 layer는 optimizer에 영향을 받지 않습니다.
0부터 max_len까지의 배열 position을 만들고, 상자를 씌웁니다. `[[0], [1], [2], ... , [max_len-1]]`. 

div_term으로 `[10000^(-2k/d) for k in torch.arange(0, d_model/2)]`와 같은 결과를 얻고, div_term의 index가 커질수록 그 값은 작아지므로, 정해진 배열인 position과 곱해져
phase(`phase = position * div_term , phase.shape = (max_len, d_model/2)`)를 만든다는 것을 생각해볼 때, 인접한 index 간 phase 차이는 3번째 축(d_model)에서 깊은 위치에 있을수록 작습니다. 

pe.shape는 (max_len, 1, d_model)이 됩니다. x의 길이만큼 slicing하여 x와 더해주고 10%의 dropout을 하여 positional encoding을 구현합니다.
![image](https://user-images.githubusercontent.com/59644774/115111188-09ad0700-9fba-11eb-82c5-1d2ea3682fee.png)

가로축은 embedding dim, 세로축은 token position입니다.
 
`generate_square_subsequent_mask` upper True triangle 행렬을 생성하고 transpose하여 lower True triangle로 변환합니다. 그 후 요소가 True이면 0, False이면 -inf 로 변환합니다. 미래에 대한 
정보를 얻지 않도록 masking을 구현했습니다.


```python
class LineCNNTransformer(torch.nn.Module):
    """Process the line through a CNN and process the resulting sequence with a Transformer decoder"""

    def __init__(
        self,
        data_config: Dict[str, Any],
        args: argparse.Namespace = None,
    ) -> None:
        super().__init__()
        self.data_config = data_config
        self.input_dims = data_config["input_dims"]
        self.num_classes = len(data_config["mapping"])
        inverse_mapping = {val: ind for ind, val in enumerate(data_config["mapping"])}
        self.start_token = inverse_mapping["<S>"]
        self.end_token = inverse_mapping["<E>"]
        self.padding_token = inverse_mapping["<P>"]
        self.max_output_length = data_config["output_dims"][0]
        self.args = vars(args) if args is not None else {}

        self.dim = self.args.get("tf_dim", TF_DIM)
        tf_fc_dim = self.args.get("tf_fc_dim", TF_FC_DIM)
        tf_nhead = self.args.get("tf_nhead", TF_NHEAD)
        tf_dropout = self.args.get("tf_dropout", TF_DROPOUT)
        tf_layers = self.args.get("tf_layers", TF_LAYERS)

        # Instantiate LineCNN with "num_classes" set to self.dim
        data_config_for_line_cnn = {**data_config}
        data_config_for_line_cnn["mapping"] = list(range(self.dim))
        self.line_cnn = LineCNN(data_config=data_config_for_line_cnn, args=args)
        # LineCNN outputs (B, E, S) log probs, with E == dim

        self.embedding = torch.nn.Embedding(self.num_classes, self.dim)
        self.fc = torch.nn.Linear(self.dim, self.num_classes)

        self.pos_encoder = PositionalEncoding(d_model=self.dim)

        self.y_mask = generate_square_subsequent_mask(self.max_output_length)

        self.transformer_decoder = torch.nn.TransformerDecoder(
            torch.nn.TransformerDecoderLayer(
                d_model=self.dim,
                nhead=tf_nhead,
                dim_feedforward=tf_fc_dim,
                dropout=tf_dropout
            ),
            num_layers=tf_layers,
        )

        self.init_weights()  # This is empirically important

    def init_weights(self):
        initrange = 0.1
        self.embedding.weight.data.uniform_(-initrange, initrange)
        self.fc.bias.data.zero_()
        self.fc.weight.data.uniform_(-initrange, initrange)
```
LineCNN 클래스의 결과로부터 바로 문자 예측을 하지 않고, output_dim을 tf_dim(256)으로 하여, 그 결과를 transformer에 넣습니다. 따라서 LineCNN이 embedding역할을 하게 됩니다.



```python
    def encode(self, x: torch.Tensor) -> torch.Tensor:
        """
        Parameters
        ----------
        x
            (B, H, W) image

        Returns
        -------
        torch.Tensor
            (Sx, B, E) logits
        """
        x = self.line_cnn(x)  # (B, E, Sx)
        x = x * math.sqrt(self.dim)
        x = x.permute(2, 0, 1)  # (Sx, B, E)
        x = self.pos_encoder(x)  # (Sx, B, E)
        return x

    def decode(self, x, y):
        """
        Parameters
        ----------
        x
            (B, H, W) image
        y
            (B, Sy) with elements in [0, C-1] where C is num_classes

        Returns
        -------
        torch.Tensor
            (Sy, B, C) logits
        """
        y_padding_mask = (y == self.padding_token)
        y = y.permute(1, 0)  # (Sy, B)
        y = self.embedding(y) * math.sqrt(self.dim)  # (Sy, B, E)
        y = self.pos_encoder(y)  # (Sy, B, E)
        Sy = y.shape[0]
        y_mask = self.y_mask[:Sy, :Sy].type_as(x)
        output = self.transformer_decoder(tgt=y, memory=x, tgt_mask=y_mask, tgt_key_padding_mask=y_padding_mask)  # (Sy, B, E)
        output = self.fc(output)  # (Sy, B, C)
        return output
```
`encode` Input Image를 LineCNN에 통과시켜 embedding하고 차원에 맞게 scaling 후 positional encoding합니다. 

`decode` target(=y)을 embedding합니다. 이때 `torch.Embedding(self.num_classes, self.dim)`을 이용하는데, token 하나( mapping = {'A': 0, ..}에서 0)당 self.dim 길이의 텐서로 변환되므로 (Sy,B)의 shape을 갖는 y의 embedding 후의 shape는 (Sy, B, E)가 됩니다. (이때 E = self.dim) 
y에 positional encoding 이후 decoderlayer에 통과시키고 fc layer로 num_classes에 대해 결과를 얻어냅니다.

![image](https://user-images.githubusercontent.com/59644774/115113946-c78ac200-9fc7-11eb-9652-5826f03c722f.png)


```python
    def forward(self, x: torch.Tensor, y: torch.Tensor) -> torch.Tensor:
        """
        Parameters
        ----------
        x
            (B, H, W) image
        y
            (B, Sy) with elements in [0, C-1] where C is num_classes

        Returns
        -------
        torch.Tensor
            (B, C, Sy) logits
        """
        x = self.encode(x)  # (Sx, B, E)
        output = self.decode(x, y)  # (Sy, B, C)
        return output.permute(1, 2, 0)  # (B, C, Sy)

    def predict(self, x: torch.Tensor) -> torch.Tensor:
        """
        Parameters
        ----------
        x
            (B, H, W) image

        Returns
        -------
        torch.Tensor
            (B, Sy) with elements in [0, C-1] where C is num_classes
        """
        B = x.shape[0]
        S = self.max_output_length
        x = self.encode(x)  # (Sx, B, E)

        output_tokens = (torch.ones((B, S)) * self.padding_token).type_as(x).long()  # (B, S)
        output_tokens[:, 0] = self.start_token  # Set start token
        for Sy in range(1, S):
            y = output_tokens[:, :Sy]  # (B, Sy)
            output = self.decode(x, y)  # (Sy, B, C)
            output = torch.argmax(output, dim=-1)  # (Sy, B)
            output_tokens[:, Sy] = output[-1:]  # Set the last output token

        # Set all tokens after end token to be padding
        for Sy in range(1, S):
            ind = (output_tokens[:, Sy - 1] == self.end_token) | (output_tokens[:, Sy - 1] == self.padding_token)
            output_tokens[ind, Sy] = self.padding_token

        return output_tokens  # (B, Sy)
```


    
```python
    @staticmethod
    def add_to_argparse(parser):
        LineCNN.add_to_argparse(parser)
        parser.add_argument("--tf_dim", type=int, default=TF_DIM)
        parser.add_argument("--tf_fc_dim", type=int, default=TF_DIM)
        parser.add_argument("--tf_dropout", type=float, default=TF_DROPOUT)
        parser.add_argument("--tf_layers", type=int, default=TF_LAYERS)
        parser.add_argument("--tf_nhead", type=int, default=TF_NHEAD)
        return parser
