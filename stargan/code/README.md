## ```main.py```

- Train을 수행하여 main.py를 실행해주면 --인자로 모든 인자들을 받아와 config 됨

```python

if config.mode == 'train':
  if cofig.dataset in ['CelebA', 'RaFD']:
    solver.train()
if config.mode == 'test':
  if cofig.dataset in ['CelebA', 'RaFD']:
    solver.train()
```

train 과 test는 ```solver.py``` 내에 저장되어있다.

#### 변수 설명

- ```c_dim``` :데이터셋 에서 사용할 특성의 수
- ```image_size```: [1,3,256,256]
- ```g_conv_dim```: Generator구조의 첫번째 layer의 filter 수 (default = 64)
- ```g_repeat_num```: Generator구조의 Residual Block의 수(default = 6)
- ```d_conv_dim```: Discriminator구조의 첫번째 layer의 filter 수 (default = 64)
- ```d_repeat_num```: Discriminator구조의 Output layer를 제외한 convolution layer의 수 (default = 6)
- ```lambda_gp```: adversarial loss를 구하는데 사용되는 gradient penalty의 값 (default = 10)
- ```num_iters```: 학습 과정에서의 몇번의 iteration을 돌 것인지(default = 200000)
- ```n_critic```: Discriminator가 몇 번 업데이트 되었을 때 Generator를 한번 update시킬 것인지
- ```selected_attrs```: CelebA 데이터셋에서 사용할 특성들 ('Black Hair, Blond Hair, Brown Hair, Male, Young')
- ```test_iters```: 모델 테스트를 위해 학습된 모델을 몇번쨰 step에서 가져 올 것인지
- ```num_workers```: 몇개의 CPU 코어에 할당할 것인지
- ```mode```: 모델을 Train으로 할 것인지 test로 할 것인지


## ```model.py```

![image](https://user-images.githubusercontent.com/72767245/118234112-dd5bac00-b4cd-11eb-8778-86f259bc8d75.png)


#### ```ResidualBlock```
- Generator의 Bottleneck부분에 사용되는 ```residual block```
```python
class ResidualBlock(nn.Module):
  def __init__(self, dim_in, dim_out):
    super(ResidualBlock, self).__init__()
```

- layers [conv-instanceNorm-ReLU-Conv-InstanceNorm]
```python
self.main = nn.Sequential(
  nn.Conv2d(dim_in, dim_out, kernel_size = 3, stride = 1, padding = 1, bias = False),
  nn.InstanceNorm2d(dim_out, affine = True, track_running_stats = True),
  nn.ReLU(inplace = True),
  nn.Conv2d(dim_out, dim_out, kernel_size = 3, stride = 1, padding = 1, bias = False),
  nn.InstanceNorm2d(dim_out, affine = True, track_running_stats = True))
```
```python
def forward(self,x):
  return x + self.main(x)
```

#### ```Generator```
- 첫번째 Convolution layer의 output dimension인 ```conv_dim```, domain label 수 ```c_dim```, Residual Block의 수 ```repeat_num```
```python
class Generator(nn.Module):
  def __init__(self, conv_dim = 64, c_dim = 5, repeat_num = 6):
    super(Generator, self).__init__()
```

layers이라는 list 안에 layers.append을 해줌
- 첫 conv2d(3 depth + c_dim) 해주어야 함 (depth: 8 -> 64)

```python
layers = []
layers.append(nn.Conv2d(3+c_dim, conv_dim, kernel_size = 7, stride = 1, padding = 3, bias = False))
layers.append(nn.InstanceNorm2d(conv_dim, affine = True, track_running_stats = True))
layers.append(nn.ReLU(inplace = True))
```
**DownSampling Layer**

```python
curr_dim = conv_dim #64dim

# [conv-instanceNorm-ReLU] 2번 반복
for i in range(2): 
  layers.append(nn.Conv2d(curr_dim, curr_dim*2, kernel_size=4, stride=2, padding=1, bias=False))
  layers.append(nn.InstanceNorm2d(curr_dim*2, affine=True, track_running_stats=True))
  layers.append(nn.ReLU(inplace=True))
  curr_dim = curr_dim*2
```

- 마지막에 self.main = nn.Sequential(*layers) 로 합쳐줌
```python
layers.append(nn.Conv2d(curr_dim, 3, kernel_size = 7, stride = 1, padding = 3, bias = False)
layers.append(nn.Tanh())
self.main = nn.Sequential(*layers)
```


#### ```Discriminator```
- Input Layer
```python
class Discriminator(nn.Module):
  def __init__(self, image_size = 128, conv_dim = 64, c_dim = 5, repeat_num = 6):
    super(Discriminator, self).__init__()
    layers = []
    layers.append(nn.Conv2d(3, conv_dim, kernel_size = 4, stride = 2, padding  = 1))
    layers.append(nn.LeakyReLU(0.01))
```
- Hidden Layer
```python    
curr_dim = conv_dim
for i in range(1, repeat_num):
  layers.append(nn.Conv2d(curr_dim, curr_dim * 2, kernel_size = 4, stride = 2, padding=1))
  layers.append(nn.LeakyReLU(0.01))
  curr_dim = curr_dim * 2
```
- Output Layer
```python
  kernel_size = int(image_size / np.power(2, repeat_num))
  self.main= nn.Sequential(*layers)
  self.conv1 = nn.Conv2d(curr_dim, 1, kernel_size =3, stride= 1, padding= 1, bias = False)
  self.conv2 = nn.Conv2d(curr_dim, c_dim, kernel_size = kernel_size, bias = False)
```
- forward 함수
```python
def forward(self, x):
  h = self.main(x)
  out_src = self.conv1(h)
  out_cls = self.conv2(h)
  return out_src, out_cls.view(out_cls.size(0), out_cls.size(1))
```
## ```solver.py```


### ---- TRAIN() 함수 전까지 ----

- Solver 클래스는 nn.Module을 상속 받지 않음
- Solver 객체를 호출할 때에는 매개변수에 celeb_loader, rafd_loader, config 를 넘겨줌

```python
class Solver(object):
  def __init__(self, celeba_loader, rafd_loader, config):
    # config 파일 내에서 값들 초기화
    ...
    # Build the model and tensorboard
    self.build_model()
    if self.use_tensorboard():
      self.build_tensorboard()
```
- build_model함수에서는 Generator와 Discriminator를 만듦

```python
    # Create a generator and a discriminator
    def build_model(self):
        # dataset이 하나일 때, self.c_dim
        if self.dataset in ['CelebA', 'RaFD']:
            self.G = Generator(self.g_conv_dim, self.c_dim, self.g_repeat_num)
            self.D = Discriminator(self.image_size, self.d_conv_dim, self.c_dim, self.d_repeat_num) 
        # dataset이 두개 다 일 때, self.c_dim + self.c_dim + 2
        elif self.dataset in ['Both']:
            self.G = Generator(self.g_conv_dim, self.c_dim+self.c2_dim+2, self.g_repeat_num)   # 2 for mask vector.
            self.D = Discriminator(self.image_size, self.d_conv_dim, self.c_dim+self.c2_dim, self.d_repeat_num)
            
        # main.py에서 learning rate와 beta 값에 대한 기본 정보를 확인할 수 있음
        self.g_optimizer = torch.optim.Adam(self.G.parameters(), self.g_lr, [self.beta1, self.beta2])
        self.d_optimizer = torch.optim.Adam(self.D.parameters(), self.d_lr, [self.beta1, self.beta2])
        
        self.print_network(self.G, 'G')
        self.print_network(self.D, 'D')
            
        self.G.to(self.device)
        self.D.to(self.device)
```

- print_network() 함수는 인자로 모델과 모델의 이름을 전달받아 모델의 네트워크 정보를 출력하는 역할을 함
- for 문에서는 model의 모든 파라미터의 원소 수를 numel()함수로 구해 num_params에 더한다

```python
def print_network(self, model, name):
  num_params = 0
  for p in model.parameters():
    num_params += p.numel()
  print(model)
  print(name)
  print("The number of parameters: {}" .format(num_params))
```

- restore_model()함수는 이전에 학습하여 저장된 모델을 불러오는 역할

```python
def restore_model(self, resume_iters):
  print('Loading the trained models from step {}...'.format(resume_iters))
  G_path = os.path.join(self.model_save_dir, '{}-G.ckpt'.format(resume_iters))
  D_path = os.path.join(self.model_save_dir, '{}-D.ckpt'.format(resume_iters))
  
  self.G.load_state_dict(torch.load(G_path, map_location=lambda storage, loc: storage))
  self.D.load_state_dict(torch.load(D_path, map_location=lambda storage, loc: storage))
```

build_tensorboard() 함수에서는 logger.py에 정의된 Logger 클래스 객체를 생성

- tensorboard: 데이터를 시각화해서 그래프로 보기 좋게 표현해주는 Tool 

```python
  def build_tensorboard(self):
        from logger import Logger
        self.logger = Logger(self.log_dir)
```
- update_lr() 함수에서는 g_optimizer와 d_optimizer에서의 learning rate를 업데이트

```python
def update_lr(self, g_lr, d_lr):
  for param_group in self.g_optimizer.param_groups:
    param_group['lr'] = g_lr
  for param_group in self.d_optimizer.param_groups:
    param_group['lr'] = d_lr
```
- reset_grad()에서는 g_optimizer와 d_oprimizer의 gradient를 0으로 reset
```python
def reset_grad(self):
  self.g_optimizer.zero_grad()
  self.d_optimizer.zero_grad()
```
- denorm 은 out의 모든 원소들을 [0,1] 범위로 만들어서 반환함

```python
def denorm(self, x):
  out = (x+1)/2
  return out.clamp_(0,1)
```
- gradient_penalty()

```python
def gradient_penalty(self, y, x):
  weight = torch.ones(y.size()).to(self.device)
  dydx = torch.autograd.grad(outputs = y, inputs = x, grad_outputs = weight, 
                      retain_graph = True, create_graph = True, only_inputs = True)[0]
  dydx = dydx.view(dydx.size(0), -1)
  dydx_l2norm = torch.sqrt(torch.sum(dydx**2, dim = 1))
  return torch.mean((dydx_l2norm-1) **2)
```
- create_labels()

```python
    def create_labels(self, c_org, c_dim=5, dataset='CelebA', selected_attrs=None):
        """Generate target domain labels for debugging and testing."""
        # Get hair color indices.
        if dataset == 'CelebA':
            hair_color_indices = []
            for i, attr_name in enumerate(selected_attrs):
                if attr_name in ['Black_Hair', 'Blond_Hair', 'Brown_Hair', 'Gray_Hair']:
                    hair_color_indices.append(i)

        c_trg_list = []
        for i in range(c_dim):
            if dataset == 'CelebA':
                c_trg = c_org.clone()
                if i in hair_color_indices:  # Set one hair color to 1 and the rest to 0.
                    c_trg[:, i] = 1
                    for j in hair_color_indices:
                        if j != i:
                            c_trg[:, j] = 0
                else:
                    c_trg[:, i] = (c_trg[:, i] == 0)  # Reverse attribute value.
            elif dataset == 'RaFD':
                c_trg = self.label2onehot(torch.ones(c_org.size(0))*i, c_dim)

            c_trg_list.append(c_trg.to(self.device))
        return c_trg_list
```
- c_org: 한 batch를 가져왔을 때 batch_size개의 이미지들의 **실제 도메인 레이블을 담고 있는 tensor**
```python
# batch size = 16
print(c_org)
-> tensor([2, 5, 2, 2, 3, 6, 0, 0, 4, 4, 6, 3, 4, 4, 5, 5])
```
