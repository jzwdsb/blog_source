---
layout: post
category: matlab
date: 2018-02-03
title: matlab与神经网络
description: matlab中神经网络工具包的使用
tags: [matlab, 神经网络]
---

## 单层感知器神经网络

1. 以 newp 创建感知器神经网络

    根据要解决的问题，确定输入向量的取值范围和维数，网络层的神经元数目．传输函数和学习函数；然后以单层感知器神经网络的创建函数 newp 创建网络
2. 以 train 训练创建网络

    构造训练样本集，确定每个样本的输入向量和目标向量，调用函数 train 对网络进行训练，并根据训练情况决定是否要调整训练参数，已得到满足误差性能指标的神经网络，然后存储
3. 以 sim 对训练后的网络进行仿真

    构造测试样本集，加载训练后的网络，调用函数 sim, 以测试样本集进行仿真，查验网络的性能

　　重要的感知器神经网络函数有: newp, train 和 sim. 除此之外还有 init, trainc, dotprod, netsum, mae, plotpc, plotpv. etc. 

### 示例代码

~~这高亮TM的什么鬼~~

```matlab
%% 初始化感知器网络
pr = [-1 1; -1 1];      % 设置感知器网络的输入向量的每个元素的值域
net = newp(pr, 1)       % 定义感知器神经网络
% net.layers{1}.transferFcn = 'hardlims';

%% 训练感知器网络
p = [0.5 -1;1 0.5; -1 0.5;-1 -1]';  % 输入向量
t = [0 1 1 0];                      % 目标向量
[net, tr] = train(net, p, t);       % 训练感知器网络

% 存储训练后网络
save net net

%% 开始仿真
clear all;
load net net

%% 网络仿真
p = [0.5 -1;1 0.5;-1 0.5;-1 -1]';    % 输入向量
a = sim(net, p)                      % 仿真结果

%% 绘制网络的分类结果及分类线
v = [-2 2 -2 2];                % 设置坐标的范围
plotpv(p, a, v);                % 绘制分类结果
plotpc(net.IW{1}, net.b{1});    % 绘制分类线


```

## 线性神经网络

1. 以 newlin 创建神经网络

    根据问题确定输入输入向量的取值范围和维数，网络层的神经元数目；然后以线性神经网络的创建函数 newlin 创建网络
2. 以 train 训练创建网络，或以 adapt 自适应调整权值和阈值．

    构造训练样本集，确定每个样本的输入向量和目标向量，调用函数 train 对网络进行训练(或以 adapt 自适应调整权值和阈值)，并根据训练的情况，决定是否调整训练参数，以得到满足误差性能指标的神经网络．
3. 若以 newlind 设计线性神经网络，则不需要训练
4. 用 sim 对训练后的网络仿真

    如果问题需要网络的仿真结果，则需要构造测试样本集，加载训练后网络，调用 sim 以得到网络的仿真结果
5. 注意：有些应用中，自适应线性神经网络的输出不是取自线性神经网络的输出，而是目标相应 t 与模拟输出量 a 的误差 e = t - a.

　　涉及到的线性神经网络函数有， newlin, newlind, adapt, train, sim. 除此之外还有 init, mse. etc

### 示例代码

~~难道 rouge 不能设置纯文本吗~~

```plain
%% 与逻辑实现 matlab 仿真
clear all;

%% 设计线性神经网络
p = [0 0; 0 1; 1 0; 1 1]';  % 输入向量
t = [0 0 0 1];              % 目标向量
net = newlind(p, t);        % 设计线性神经网络
w = net.IW{1};              % 输出训练后的权值
b = net.b{1}                % 输出训练后的阈值

%% 仿真
a = sim(net, p);            % 输出仿真结果
y = a > 0.5;                % 将模拟仿真结果转换成数字量

%% 噪声对消
clear all;
time = 0.01:0.01:10;                    % 时间变量
noise = (rand(1, 1000) - 0.5) * 4;      % 随机噪声
input = sin(time);                      % 信号
p = noise;                              % 将噪声作为 ADALINE 的输入向量
t = intput + noise                      % 将噪声 + 信号作为目标向量

%% 创建线性神经网络
net = newlin([-1 1], 1, 0, 0.0005);

% 调整(训练)
net.adaptParam.passes = 70;
[net, y, output] = adapt(net, p, t);    % output 为网络调整过程中的误差

%% 绘制信号，迭加随机噪声的信号，输出信号的波形
hold on;
% 绘制信号的波形
subplot(3, 1, 1);
plot(time, input, 'b');
xlabel('t', 'position', [10.5 -1]);
ylabel('信号波形 sin(t)', 'fontsize', 8);
% 绘制迭加随机噪声的信号波形
subplot(3, 1, 2);
plot(time, t, 'm');
xlabel('t, 'position', [10.5 -5]);
ylabel('随机噪声波形 sin(t) + noise(t)', 'fontsize', 8);
% 绘制输出信号的波形
subplot(3, 1, 3);
plot(time, output, 'y');
xlabel('t', 'position', [10.5 -2]);
ylabel('输出信号波形 y(t)', 'fontsize', 8);
hold off;
```

## BP 神经网络

### 网络设计的几大方面

　　BP 网络的设计主要包括 输入层，隐层，输出层及各层之间的传输函数几方面，

1. 网络的层数，两层的 BP 可以实现任意非线性映射，一般不超过两层
2. 输入层的节点数，取决于输入向量的维数
3. 输出层的节点数，取决于数据类型和表示该类型所需的数据大小，使用二进制数表示时，可以用 n 位二进制数表示，编码可以用 $$\log_{2}{n}$$表示
4. 隐层的节点数，根据前人经验，可以参照以下公式设计

    　　　　　　　　　　　　$$n = \sqrt{n_i + n_o} + a$$<br>
    $$n$$ 为隐层节点数；$$n_i$$ 为输入节点数，$$n_o$$ 为输出节点数；$$a$$ 为 1 到 10 的常数
5. 传递函数，通常采用 S(sigmoid)型函数

    　　　　　　　　　　　　$$f(x) = \frac{1}{1 + e^{-x}}$$
6. 训练方法及参数选择

　　BP 神经网络用到的函数
1. newff 创建 BP 神经网络
2. train 训练
3. sim   仿真

### 示例代码

~~算了，至少有颜色~~

```matlab
%%% 两类模式的分类
clear all;

p = [1 2; -1 1; -2 1; -4 0]';   % 输入向量
t = [0.2 0.8 0.8 0.2];          % 目标向量

%% 创建 BP 网络和定义训练函数
net = newff([-1 1; -1 1], [5 1], {'logsig' 'logsig'}, 'traingd');
net.trainParam.goal = 0.001;
net.trainParam.epochs = 5000;

%% 训练神经网络
[net, tr] = train(net, p, t);

%% 输出训练后的权值与阈值
iw1 = net.IW{1};
b1 = net.b{1};
lw2 = net.IW{2};
b2 = net.b{2};

%% 存储训练号的神经网络
save net net;

load net net;
p = [1 2; -1 1; -2 1; -4 0];    % 测试输入向量
a2 = sim(sim(net, p);           % 仿真输出结果
a2 = a2 > 0.5;                  % 根据判决门限，输出分类结果

%%% 数字识别 BP 神经网络
clear all;

%% 生成输入向量和目标向量
for kk = 0: 89
    pl = ones(16, 16);                   % 初始化 16 * 16 的二值图像像素值(全白)
    m = strcat(int2str(kk), '.bmp');     % 形成训练样本图像的文件名(0 ~ 89.bmp)
    x = imread(m, 'bmp');                % 读入训练样本图像文件
    bw = im2bw(x, 0.5);                  % 将读入的训练样本图像转换为二值图像
    [i, j] = find(b2 == 0);              % 寻找二值图像中像素值为 0 (黑) 的行号和列号
    imin = min(i);                       % 寻找二值图像中像素值为 0 (黑) 的最小行号
    imax = max(i);                       % 寻找二值图像中像素值为 0 (黑) 的最大行号
    jmin = min(j);                       % 寻找二值图像中像素值为 0 (黑) 的最小列号
    jmax = max(j);                       % 寻找二值图像中像素值为 0 (黑) 的最大列号
    bwl = bw(imin:imax, jmin:jmax);      % 截取图像像素值为 0 的最大矩形区域
    rate = 16 / max(size(bw1));          % 计算截取图像转换为 16 * 16 的二值图像的缩放比例
    bw1 = imresize(bw1, rate);           % 将截取图像转换为 16 * 16 的二值图像
                                         % (由于缩放比例大多数情况下不是 16 的倍数)
    [i, j] = size(bw1);                  % 转换图像的大小
    i1 = round((16 - i) / 2);            % 计算转换图像与标准 16 * 16 的图像的左边界差
    j1 = round((16 - i) / 2);            % 计算转换图像与标准 16 * 16 的图像的上边界差
    p1(i1 + 1: i1 + i, j1 + 1: j1 + j) = bw1 %将截取图像转换为标准的 16 * 16 的图像
    p1 = -1 .* p1 + ones(16, 16);        % 反色处理

    % 以图像数据形成神经网络输入向量
    for m = 0 : 15
        p(m * 16 + 1: (m + 1) * 16, kk + 1) = p1(1: 16, m + 1);
    end

    % 形成神经网络目标向量
    switch kk
        case {0, 10, 20, 30, 40, 50, 60, 70, 80, 90}    % 数字 0
            t(kk + 1) = 0;
        case {1, 11, 21, 31, 41, 51, 61, 71, 81, 91}    % 数字 1
            t(kk + 1) = 1;
        case {2, 12, 22, 32, 42, 52, 62, 72, 82, 92}    % 数字 2
            t(kk + 1) = 2;
        case {3, 13, 23, 33, 43, 53, 63, 73, 83, 93}    % 数字 3
            t(kk + 1) = 3;
        case {4, 14, 24, 34, 44, 54, 64, 74, 84, 94}    % 数字 4
            t(kk + 1) = 4;
        case {5, 15, 25, 35, 45, 55, 65, 75, 85, 95}    % 数字 5
            t(kk + 1) = 5;
        case {6, 16, 26, 36, 46, 56, 66, 76, 86, 96}    % 数字 6
            t(kk + 1) = 6;
        case {7, 17, 27, 37, 47, 57, 67, 77, 87, 97}    % 数字 7
            t(kk + 1) = 7;
        case {8, 18, 28, 38, 48, 58, 68, 78, 88, 98}    % 数字 8
            t(kk + 1) = 6;
        case {9, 19, 29, 39, 49, 59, 69, 79, 89, 99}    % 数字 2
            t(kk + 1) = 9;
    end
end
save traindata p t                     % 储存形成的训练样本集(输入向量和目标向量)

%% 构造 BP 神经网络,并根据训练样本集形成的输入矢量和目标向量,对 BP 网络进行训练
clear all;
load traindata p t;                    % 加载训练样本集

% 创建 BP 网络
pr(1: 256, 1) = 0;
pr(1: 256, 2) = 1;
net = newff(pr, [25 1], {'logsig' 'purelin'}, 'traingdx', 'learngdm');

% 设置训练参数和训练 BP 网络
net.trainParam.epochs = 2500;
net.trainParam.goal = 0.001;
net.trainParam.show = 10;
net.trainParam.lr = 0.05;
net = train(net, r);

save trainnet net;

%% 仿真神经网络
clear all;
p(1:256, 1) = 1;
p1 = ones(16, 16);                     % 初始化 16 * 16 的二值图像像素值(全白)

load trainnet net;
test = input('input a test image', 's');
x = imread(test, 'bmp');
b2 = im2bw(test, 0.5):                 % 将读入图像转换为二值图像
[i, j] = find(bw == 0);
imin = min(i);
imax = max(i);
jmin = min(j);
jmax = max(j);

bw1 = bw(imin: imax, jmin: jmax);
rate = 16 /max(size(bw1));
bw1 = imresize(bw1, rate);
[i, j] = size(bw1);
i1 = round((16 - i) / 2);
j1 = round((16 - i) / 2);
p1(i1 + 1: i1 + i, j1 + 1: j1 + j) = bw1;
p1 = -1 .* p1 + ones(16, 16);
for m = 0: 15
    p(m * 16 + 1: (m + 1) * 16, 1) = p1(1:16, m+1);         % 形成测试样本输入向量
end

[a, Pf, Af] = sim(net, p);           % 网络仿真
imshow(p1);                          % 显示测试样本图像
a = round(a);                        % 输出识别结果

```

<!--
> 　　那一天我二十一岁，在我一生的黄金时代，我有好多奢望．我想爱，想吃，还想在一瞬间变成天上半明半暗的云，后来我才知道，生活就是个缓慢受锤的过程，人一天天老下去，奢望也一天天消失，最后像挨了锤的牛一样．可是我过二十一岁生日时没有预见到这一点．我觉得自己会永远生猛下去，什么也锤不了我．
-->