# 使用MATLAB的polyfit进行刹车伺服电机扭矩与刹车扭矩的曲线关系拟合

某型无人机使用了伺服电机上转子轴固定一10cm左右刹车线带动刹车机构进行刹车动作，这时就需要知道伺服电机输出多少扭矩使刹车机构产生摩擦从而生成多大刹车扭矩的关系。

目前可以使用excel表格进行曲线关系拟合，也可以使用matlab的polyfit函数进行函数曲线拟合做到同样的效果。

## 示例代码：

```matlab
%自变量x：刹车舵机输出扭矩千分比
x=[550;
    450;
    350;
    250;
    200];

%因变量y：刹车扭矩
y=[25 23 24;
    20 20 20.5;
    9.1 9.6 9.7
    7.5 5.8 6;
    2.5 2.4 1.4];

%mean中1代表纵向求平均，2意味着横向求平均
y_avr=mean(y,2);
fitres=polyfit(x,y_avr,1);
fitcur=polyval(fitres,x);

plot(x,y_avr,'-o',x,fitcur,'--');
legend('data average result','fitting result','location','northwest');
str=sprintf('y=%.5ex+%.5e',fitres);
text(x(3),fitcur(3),strcat('\leftarrow',str),'Fontsize',12);

```

## 效果：

[![pPfrMcj.png](https://z1.ax1x.com/2023/09/16/pPfrMcj.png)](https://imgse.com/i/pPfrMcj)
