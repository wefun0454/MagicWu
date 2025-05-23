# circos图的绘制

    绘制circos图可以使用circos软件在服务器中运行，也可以使用TBtools，两者出图没有本质的区别，在此介绍对小白更友好，操作更简单的TBtools，但是一些文件的获得依旧需要依靠服务器.  
    教程来源主要为：https://www.yuque.com/cjchen/hirv8i/b88df7179632c0ee5f86a4b74a58f933  
    此外含有大量个人经验，仅看本教程完全可以完成全部工作

## 1.准备文件  
原始文件为chr.fasta（如果有除了染色体之外的序列，需要全部删去），all_chr.gff（重复序列注释文件）以及Msin.chr.gff（基因组注释文件）  
然后准备文件（*表示必须）  
1）*染色体长度文件  
2）基因密度文件  
3）TE密度文件  
4）COPIA密度文件  
5）GYPSY密度文件  
6）GC含量密度文件  
7）共线性文件  

**1）染色体长度文件**

TBtools->sequence toolkits->fasta tools->fasta stater   
第一栏拖入chr.fasta，设置输出文件路径（需指定文件名），运行，提取出第一列及第四列中的数据，即得到染色体长度文件  
也可以使用samtools和bedtools来完成，后面的文件也都需要这两个软件，图省事可以使用conda安装  
```
wget https://github.com/arq5x/bedtools2/archive/v2.30.0.tar.gz
tar -zxvf v2.30.0.tar.gz
cd bedtools2
make
```
```
wget https://github.com/samtools/samtools/releases/download/1.13/samtools-1.13.tar.bz2
tar -xvjf samtools-1.13.tar.bz2 
cd samtools-1.13
./configure
make
```
安装好之后  
```
samtools dict chr.fasta > out
cat out | sed 's/:/\t/g'|cut -f 3,5 >genome.chr.ln
```  

- cat out：读取前一步生成的字典文件  
- sed 's/:/\t/g'：将行中的冒号替换为制表符（TAB），将信息分隔成多列  
- cut -f 3,5：提取第三列（染色体名称）和第五列（染色体长度）  
- \> genome.chr.ln：'将结果输出到genome.chr.ln文件中  

**如果使用circos软件绘图需要使用以下方法**  
```
genome=tair10.genome.fa
faidx ${genome} -i chromsizes > tair10.genome.size
awk -vFS="\t" -vOFS="\t" '{print "chr","-",$1,"C"NR,"0",$2,"chr"NR}' tair10.genome.size > circos.chromosome.txt
cat circos.chromosome.txt
```

**2）基因密度文件**

```
grep -w "gene" Msin.chr.gff |awk '{print $1"\t"$4"\t"$5}'|uniq > gene.pos
```
- grep：用于在文件中查找匹配特定模式的行  
- -w：表示精确匹配整个单词gene，确保只匹配gene（而不是匹配包含gene作为部分的单词，如gene_name等）  
- awk：是一个文本处理工具，常用于提取文件的某些列  
- '{print $1"\t"$4"\t"$5}'：这是awk的输出格式，表示提取并输出第1列（染色体）、第4列（基因起始位置）和第5列（基因结束位置），并以制表符 (\t) 分隔  
- uniq：用于删除相邻的重复行。如果文件中的某些基因注释是相同的并且相邻出现，uniq可以确保只保留一个实例  

```
bedtools makewindows -g genome.chr.ln -w 100000 > 100K.genome.3col
```
- bedtools makewindows：生成固定大小的滑窗（窗口）  
- -g genome.chr.ln：指定包含染色体名称和长度的文件，即前一步生成的genome.chr.ln文件  
- -w 100000：指定滑窗的大小为100,000个碱基对（100kb）  
- \> 100K.genome.3col：将生成的滑窗信息输出到100K.genome.3col文件中  

```
bedtools intersect -a 100K.genome.3col -b gene.pos -c > gene.out
```
- bedtools intersect：比较两个基因组区间文件，并计算它们之间的重叠
- -a 100K.genome.3col：指定第一个输入文件，这里是包含100kb滑窗的文件（例如100K.genome.3col），这个文件表示基因组上的100kb窗口
- -b gene.pos：指定第二个输入文件，这里是包含基因或其他元素（如LTR、GYPSY、COPIA等）位置的文件（gene.pos）
- -c：计算每个滑窗中与gene.pos中的基因或元素重叠的数量，并将结果添加到输出文件中作为新列
- \> gene.out：将结果输出到文件gene.out中

如果需要  
```
sed -i '/^ptg/d' gene.out
```
-删除序列名中有‘ptg’的序列。这是因为有一些基因组注释文件中包含除染色体之外的基因注释，如我的染色体注释文件中就有ptg开头的序列，因此需要将不相关的统计结果删除

**3）COPIA密度  
4）GYPSY密度**  
```
awk '{print $1"\t"$4"\t"$5}'  ../JNYL/all_chr.gff  |grep "Copia" >copia.bed
bedtools intersect -a 100K.genome.3col -b COPIA.pos -c > COPIA.out
```
如果需要  
```
sed -i '/^ptg/d' COPIA.out
```

**5）TE密度文件**

TE密度文件跟重复序列注释文件关系十分密切，如果重复序列注释文件类似以下形式（Repeat Masker注释）:

    Chr5	RepeatMasker	Transposon	5635057	5638270	25165	+	.	ID=genome_TE0000001;Target=LTR_FINDER_ID=ctg54_3_ltr; 1895 5289;Class=LINE/RTE-BovB;PercDiv=2.7;PercDel=0.2;PercIns=0.3;
    Chr5	RepeatMasker	Transposon	5639304	5639382	422	-	.	ID=genome_TE0000002;Target=LTR_FINDER_ID=ctg54_3_ltr; 3093 3171;Class=LINE/RTE-BovB;PercDiv=16.5;PercDel=0.0;PercIns=0.0;

就与上面类似，或者可以直接使用注释文件，因为基本全是TE  
```
grep "Transposon" ../JNYL/all_without_trf_chr.gff > TE.gff
```
如果不是，就检索关键词，将Transposon换为TE，或者Transposon Elements等，如果实在弄不清楚，就不要这一个文件，或者尝试使用EDTA，TE Density重新注释TE，构建TE库，但这与本篇教程目的相背离，因此后面再出详细教程

**6）GC含量文件**

TBtools->sequence toolkits->fasta tools->fasta Windows stat  
输入文件为chr.fasta，窗口大小设置为10000，相邻窗口之间重叠的碱基数目设置1000，这两参数一般为默认  
指定输出文件路径，运行  
获得的文件为.GCratio文件

**7）共线性文件**

共线性文件可以使用服务器构建，也可以使用TBtools构建，在此介绍更为简单的TBtools  
TBtools->other->plugin->plugin store->sequence analyses->onestepMCScanX-superfast->install  
下载好之后onestepMCScanX-superfast位置在TBtools->other->plugin->sequence analyses->onestepMCScanX-superfast  
第一个和第三个输入文件都为chr.fasta，第二第四个文件均为Msin.chr.gff，指定输出文件路径，运行
最终.geneLinkedRegion.tab.xls文件为共线性文件  
如果要添加颜色，打开共线性文件，在最后一列填入相应的颜色，颜色为RGB格式，可在网站寻找：
https://www.rapidtables.org/zh-CN/web/color/html-color-codes.html#:~:text=%E9%A2%9C%E8%89%B2%20HTML%20/

例如：

    Chr1	10556167	10563729	Chr2	72596456	72608457
    Chr1	8479089	8487742	Chr3	8238545	8249992	"255,255,0"
    Chr1	8488084	8491727	Chr3	8256021	8262718	"255,255,0"

这样可显示chr1和chr3的共线性颜色，chr1和chr2为默认的浅灰色

使用服务器可使用jcvi建立自比对，此步看之前的教程  
从共线性的基因对，提取整合成circos的bed格式  
```
cat TNML.TNML.anchors |grep -v \#| while read line;
do
        gene_array=($line)
        gene1_pos=`fgrep ${gene_array[0]} TNML.bed|cut -f 1-3`
        gene2_pos=`fgrep ${gene_array[1]} TNML.bed|cut -f 1-3`
        bed=${gene1_pos}" "${gene2_pos}
        echo $bed >>TNML.coline_num.txt
done
```

## 2.绘制circos图
TBtools->grapics->advance circos  
第一个和第三个文件分别输入染色体长度文件和共线性文件，第二个文件不管，运行  
如果想加入其他文件，可通过show control dialog中的add添加，可修改TrackType，有条形线型等，可自己尝试，startpos和endpos中pos指position，可调整位置，不过多赘述  
比较常见要调整的为chr bar和chr label，前者调整染色体长度条形的位置，一般设置在最外层来避免看不清染色体长度，后者调整染色体名字位置，也是同理，其余参数均不需要太更改，因此不过多介绍，可以自己尝试，一般使用AI对输出的PDF文件更改更加方便  

## 3.使用circos软件绘制（该部分教程参考博客https://www.cnblogs.com/bgicollege/p/15543482.html）
由于Circos是基于perl语言开发的，所以perl语言编译器的安装是必须的（建议Perl 5.8.x及以上版本），除此之外还需要很多perl依赖模块，可以通过测试查看缺少哪些模块并安装。详细的准备要求可以见http://circos.ca/software/requirements/

### 3.1下载解压
circos的官网的下载页面为http://circos.ca/software/download/，circos-0.69-6.tgz是目前最新版的安装包，而circos-course-2017.tgz则是配套的教程数据。  
```
wget http://circos.ca/distribution/circos-0.69-6.tgz
tar xvfz circos-0.69-6.tgz
cd circos-0.69-6
```
### 3.2安装缺失依赖模块
直接运行circos，根据报错再依次安装
```
./circos -v
```
假设出现报错

    missing Font::TTF::Font
      error Can't locate Font/TTF/Font.pm in @INC (you may need to install the Font::TTF::Font module) (@INC contains: /home/walnut168/mnt/wf/program/circos-0.69-9/bin/lib /home/walnut168/mnt/wf/program/circos-0.69-9/bin/../lib /home/walnut168/mnt/wf/program/circos-0.69-9/bin /home/walnut168/perl5/lib/perl5/x86_64-linux-thread-multi /home/walnut168/perl5/lib/perl5 /home/walnut168/miniconda3/envs/wf/lib/perl5/5.32/site_perl /home/walnut168/miniconda3/envs/wf/lib/perl5/site_perl /home/walnut168/miniconda3/envs/wf/lib/perl5/5.32/vendor_perl /home/walnut168/miniconda3/envs/wf/lib/perl5/vendor_perl /home/walnut168/miniconda3/envs/wf/lib/perl5/5.32/core_perl /home/walnut168/miniconda3/envs/wf/lib/perl5/core_perl .) at (eval 35) line 1.
安装缺失依赖模块
```
cpanm Font::TTF::Font
```

其他缺失模块以此类推

一般来讲，缺少的模块，大部分安装起来都比较顺利，可能有一个除外，就是GD。GD模块是一个图像相关的源码库，它需要很多前置的依赖包和依赖软件，可部分参考（http://starsyi.github.io/2016/05/31/%E9%9D%9Eroot%E7%94%A8%E6%88%B7perl-GD%E6%A8%A1%E5%9D%97%E7%9A%84%E5%AE%89%E8%A3%85/）
由于年代过于久远，因此参考博客有一部分错误，以下将该流程重新整理一遍。    
（1）下载并编译安装libgd
```
wget https://github.com/libgd/libgd/releases/download/gd-2.3.3/libgd-2.3.3.tar.gz
tar -xzf libgd-2.3.3.tar.gz
cd libgd-2.3.3
./configure --prefix=$HOME/local
make
make install
```
编译安装好之后还需要将将 libgd 的库路径添加到环境变量中
```
export LD_LIBRARY_PATH=$HOME/local/lib:$LD_LIBRARY_PATH
export CPATH=$HOME/local/include:$CPATH
export LIBRARY_PATH=$HOME/local/lib:$LIBRARY_PATH
```
（2）下载并安装GD
```
wget https://cpan.metacpan.org/authors/id/R/RU/RURBAN/GD-2.77.tar.gz
tar -xzf GD-2.77.tar.gz
cd GD-2.77
perl Makefile.PL --lib_gd_path=$HOME/local
make
make test
make install PREFIX=~/perl5
```
安装好之后设置Perl模块路径
```
export PERL5LIB=~/perl5/lib/perl5:$PERL5LIB
```

以上所有操作完成之后运行以下命令验证GD是否安装成功
```
perl -e 'use GD;'
```
没有报错说明安装成功，进入circos/bin目录运行以下操作检验circos是否可以正常使用
```
./circos -v
```
没有报错说明安装成功

### 3.3使用circos
#### 3.3.1准备文件
**（1）.conf文件的介绍**

圈图中各个环道，是依赖一系列依赖调用的配置文件描述出来的，从底层的颜色等，到更高层，以及最终的元配置，构成了一个堆叠起来的配置文件金字塔。
Circos的配置文件后缀为.conf，内容是标记性文本格式的，类似于HTML或者XML，每个标签对代表一个配置块，起始标记为\<TAG>，终止标记为\</TAG>。配置块中的基本语法就是:

     variable = value

举一个例子，一个简单的配置块：

    <plots>
    type = line
    thickness = 2
    ……
    <plot>

每个标签对可以使用多次，或者相互嵌套。因此就像编程中的函数或者循环结构一样，有全局变量和局部变量的区分，高级的或者外层的变量对于层内的所有实例都是有效的，而层内的定义则是优先级最高的。例如，下面配置中的  fill_color = blue   对于对  \<plots>\</plots>   中所有实例都有效，除非实例内部自己用局部变量进行定义，如内部嵌套的第一个    \<plot>\</plot> 对中的  fill_color = red    ：

    <plots>
    fill_color = blue
    ……
        <plot>
        fill_color = red
        ……
        </plot>

        <plot>
        ……
        </plot>
    </plots>

一些围绕同一个主题的重要配置块，可以另外单独写到一个配置文件中，再由其他的模块导入调用，所以理论上我们可以把所有的配置写在同一个文件中，也可以把每个配置块都分开到不同的文件中写，最后统一调用。

其中最基本的配置文件都放在$PATH/circos-0.69-4/etc下面，主要是：

    Housekeeping.conf #基本框架配置，circos必不可少的配置文件，一般直接导入即可。
    fonts.conf        #字体配置文件，其中，代表字体的标签与circos/fonts文件夹下的otf或ttf文件一一对应。
    patterns.conf     #模式配置文件，其中，标签与circos/tites文件夹下的png文件一一对应，然后根据patterns.svg.conf生成相应的svg文件。
    colors.conf       #颜色配置文件，导入了colors.brewer.conf、colors.hsv.conf和colors.ucsc.conf。在circos中，颜色除了绝对路径，还有很多表达方式，如hs1，red，reds-9-seq-1等，是因为这些字符在colors配置文件中已经赋予了特定的绝对路径。
以上的基本配置大多数情况下无需调整，直接调用导入即可。导入配置的基本格式是：

    <<include etc/*.conf>> #etc/*.conf就是要导入的配置文件的相对或者绝对路径
还有一些经常用到的其他配置，可以根据自己喜好定制：

ideogram.conf  #染色体展示的配置文件，包括是否展示bands，染色体展示位置、染色体间距离、染色体标签位置等。
ticks.conf     #刻度标签配置文件，可以导入到ideogram.conf中，也可以直接导入到主配置文件中。
image.conf     #对image大小、背景颜色、输出目录、输出文件形式以及染色体在圆周上的起始位置等进行设置。
最后，这些配置文件都会导入到circos.conf，基本内容是：

    karyotype = data/karyotype/karyotype.mouse.mm9.txt #核型文件
    chromosomes_units = 1000000 #就是其他的配置文件中距离单位u的大小，以bp来衡量，一般大的基因组倾向于把一个单位的长度设置得大一些
    chromosomes_display_default = yes
    <<include mydata/ideogram.conf>>
    <<include mydata/ticks.conf>>
    ……
    <image>
    <<include etc/image.conf>>
    </image>
    <<include etc/colors_fonts_patterns.conf>>
    <<include etc/housekeeping.conf>>
配置完所有的配置文件后，最后将这个元配置文件输入Circos就可以产生结果：
```
circos -conf circos.conf
```
**（2）基因组作图文件**

1.计算染色体长度  
安装samtools  
```
wget https://github.com/samtools/samtools/releases/download/1.13/samtools-1.13.tar.bz2
tar -xvjf samtools-1.13.tar.bz2
cd samtools-1.13
./configure
make
```

使用samtools中的插件faidx  
```
faidx ../zjll/ZJ.chr.fa -i chromsizes > zj.size
awk -vFS="\t" -vOFS="\t" '{print "chr","-",$1,"C"NR,"0",$2,"chr"NR}' zj.size > zj.circos.chromosome.txt
```

也可以使用seqtk  
```
seqtk comp ../zjll/ZJ.chr.fa | grep "^chr" | awk '{print $1"\t"$2}' > zj.genome.len
awk '{print "chr\t-\t"$1"\t"$1"\t0\t"$2"\tchr"NR}' zj.genome.len > zj_karyotype.txt
#此处zj_karyotype.txt等同于上一步中的zj.circos.chromosome.txt，zj.size等同于zj.genome.len，两者均可相互替换，后不赘述
```

zj.circos.chromosome.txt结果如下：

    #染色体 -   染色体编号(ID) 图显示名称(Label)     起始   结束    颜色
    chr     -       chr1    C1      0       33953567        chr1
    chr     -       chr2    C2      0       36348176        chr2
    chr     -       chr3    C3      0       32768115        chr3
    chr     -       chr4    C4      0       22859102        chr4
    chr     -       chr5    C5      0       19899253        chr5
    chr     -       chr6    C6      0       28512749        chr6
    chr     -       chr7    C7      0       28542370        chr7
    chr     -       chr8    C8      0       36419750        chr8
    chr     -       chr9    C9      0       33812785        chr9
    chr     -       chr10   C10     0       23968779        chr10
    chr     -       chr11   C11     0       29606729        chr11
    chr     -       chr12   C12     0       38801854        chr12
    chr     -       chr13   C13     0       49412298        chr13
    chr     -       chr14   C14     0       31566911        chr14
    chr     -       chr15   C15     0       37966322        chr15
    chr     -       chr16   C16     0       43438598        chr16

2.GC含量  
生成窗口文件
```
bedtools makewindows -w 50000 -g zj.genome.len > zj.window.bed
```
计算每个窗口平均GC含量  
```
seqtk subseq ../zjll/ZJ.chr.fa zj.window.bed > zj.window.fa
seqtk comp zj.window.fa |awk '{print $1 "\t" ($4+$5)/($3+$4+$5+$6) } ' | awk -F ":|-" '{print $1"\t"$2"\t"$3"\t"$4}'> zj_gc.txt
```
3.基因条数  
```
grep -w "gene" LV.chr.gff |awk '{print $1"\t"$4"\t"$5}'|uniq > gene.bed
```
```
bedtools coverage -a genome.window.bed-b gene.bed -c > gene.out
```
4.重复序列含量
```
bedtools coverage -a genome.window.bed -b repeat.gff | awk '{print $1 "\t" $2 "\t" $3 "\t" $7}' > zj_repeat.txt
#copia、gypsy
grep "Copia"   ../JNYL/all_chr.gff |awk '{print $1"\t"$4"\t"$5}'  >copia.bed
bedtools coverage -a genome.window.bed-b copia.bed -c > copia.out
```
5.共线性link文件
```
cat ZJ.LV.anchors | grep -v \# | while read line;
do
    gene_array=($line)
    gene1_pos=$(awk -v gene="${gene_array[0]}" '$4 == gene {print $1, $2, $3}' zj.bed)
    gene2_pos=$(awk -v gene="${gene_array[1]}" '$4 == gene {print $1, $2, $3}' LV.bed)
    bed=${gene1_pos}" "${gene2_pos}
    echo $bed >> ZL.LINKS.txt
done
```
```
cat LV.LV.anchors |grep -v \#| while read line;
do
        gene_array=($line)
        gene1_pos=`fgrep ${gene_array[0]} LV.bed|cut -f 1-3`
        gene2_pos=`fgrep ${gene_array[1]} LV.bed|cut -f 1-3`
        bed=${gene1_pos}" "${gene2_pos}
        echo $bed >>LV.LINKS.txt
done
# 使用 awk 重组数据（假设原始数据每行包含6的倍数列）
awk '{
    for(i=1;i<=NF;i+=6) {
        if($(i+5)) print $i,$(i+1),$(i+2),$(i+3),$(i+4),$(i+5)
    }
}' LV.LINKS.txt > LV.LINKS.circos.txt
```

**（3）核型配置文件**

    Circos的配置中有四种单位，p, r, u, b
    p表示像素，1p表示1像素
    r表示相对大小，0.95r表示95% ring 大小。
    u表示相对chromosomes_unit的长度，如果chromosomes_unit = 1000，则1u就是千分之一的染色体长度。
    b表示碱基，如果染色体长1M，那么1b就是百万分之一的长度。

染色体图形配置文件，ideogram.conf文件基本配置如下：

    <ideogram>
        show = yes
        <spacing>
            default = 0.005r    #染色体间的空隙
        </spacing>
        thickness      = 25p    #ideograms的厚度，可以使用r（比例关系）或p（像素）作为单位
        fill           = yes    #设定ideograms是否填充颜色。填充的颜色取决于 karyotype 指定的文件的最后一列。
        radius         = 0.8r   #设定ideograms的位置
        show_label     = yes    #设定是否显示label
        label_font     = default#设定label的字体
        label_radius   = dims(ideogram,radius_outer) + 80p #设定label的位置
        label_size     = 40p    #设定label的字体大小
        label_parallel = yes    #设定label的字体方向，yes是易于浏览的方向。
    </ideogram>

刻度文件，ticks.conf，基本配置如下：

    <ticks>
        tick_label_font  = light
        radius           = dims(ideogram,radius_outer) + 10p  # 设定 ticks 的位置
        label_offset     = 5p  # 设置 ticks' label 离 ticks 的距离
        label_size       = 20p  # 设置 ticks' label 的字体大小
        multiplier       = 0.001  # 设定 ticks' label 的值的计算。将该刻度对应位置的值 * multiplier 得到能展示到圈图上的 label 值。
        color            = black  # 设定 ticks 的颜色
        thickness        = 1p  # 设定 ticks 的厚度

        <tick>
            spacing        = 50u  # 设置每个刻度代表的长度。
            size           = 12p  # 设置 tick 的长度
            show_label     = yes  # 是否显示 tick labels
        </tick>

        <tick>
            label_separation = 1p
            spacing          = 5u
            size             = 7p
            show_label       = no
        </tick>
    </ticks>

主配置文件，circos.conf，主要配置如下：

    karyotype = karyotype.txt #指定染色体文件
    chromosomes_units = 1000000 #设置长度单位，表示为1M长度的序列代表为1u
    chromosomes_display_default = yes #默认是将所有的染色体都展示出来
    ## 载入ideogram配置和刻度线配置
    show_ticks = yes # 显示刻度
    show_tick_labels = yes # 显示刻度label
    ## plots block 绘制折线图、散点图、直方图、热图和文本显示
    <plots>
    ## 绘制直方图
    <plot>
    type = histogram # histogram, line,scatter, heatmap,text
    file = data/gene_density.txt #文件路径
    # 设置直方图的位置，r1 > r0
    r1 = 0.98r
    r0 = 0.9r
    orientation = out #直方图方向，in为向内
    fill_color = blue #直方图的填充颜色
    thickness = 2p #直方图轮廓厚度为 1p。
    extend_bin = no #若bins在坐标上不相连，最好设置不要将其连接到一起。
    # 设定直方图的背景颜色
    <backgrounds>
    <background>
    color = vlgrey
    </background>
    </backgrounds>
    </plot>
    ##绘制热图
    <plot>
    type = heatmap # 指定绘图类型为heatmap
    file = data/gene_density.txt # 设定数据文件路径
    # 设定图形所处位置
    r1 = 0.88r
    r0 = 0.8r
    color = green,yellow,red # 设定热图的颜色
    scale_log_base = 1 # 设定 scale_log_base 参数
    </plot>
    ##绘制折线图
    <plot>
    type = line # 指定绘图类型为line
    file = data/gene_density.txt # 设定数据文件路径
    # 设定图形所处位置
    r1 = 0.78r
    r0 = 0.7r
    color = green # 设定颜色
    <backgrounds>
    <background>
    y1 = 0.5r
    y0 = 0r
    color = vvlgrey
    </background>
    <background>
    y1 = 1r
    y0 = 0.5r
    color = vvlred
    </background>
    </backgrounds>
    </plot>
    ##绘制散点图
    <plot>
    type = scatter # 指定绘图类型为scatter
    file = data/gene_density.txt # 设定数据文件路径
    # 设定图形所处位置
    r1 = 0.68r
    r0 = 0.6r
    glyph = circle # circle, rectangle, or triangle
    glyph_size = 4p # 点大小
    color = green # 设定颜色
    <backgrounds>
    <background>
    color = vvlgrey
    </background>
    </backgrounds>
    </plot>
    ##显示文本
    <plot>
    type = text #设置绘图类型为文本
    file = data/text.gene.txt # 数据文件路径
    color = black #文字颜色
    # 显示在图形中的位置
    r1 = 0.58r
    r0 = 0.45r
    label_font = default # 标签的字体
    label_size = 20p # 标签大小
    rpadding = 1p # 文字边缘的大小
    # 设置是否需要在 label 前加一条线，用来指出 lable 的位置。
    show_links = yes
    link_dims = 0p,2p,5p,2p,2p
    link_thickness = 2p
    link_color = red
    </plot>
    </plots>
    ##links画连接线
    <links>
    <link>
    file = data/links.genePair.txt #link文件
    color = blue #设置颜色
    radius = 0.6r #设置 link 曲线的半径
    bezier_radius = 0r #设置贝塞尔曲线半径，该值设大后曲线扁平。
    thickness = 2 #设置 link 曲线的厚度
    ## ribbon = yes #将线条连接改成带状连接
    ######rules指定线条的颜色，
    <rules>
    <rule>
    condition = var(chr1) eq "Chr1" #共线性文件中，左边是Chr1的，指定颜色。var(chr1)这个不用修改，指的是file里左侧第一个，后面的“Chr1”根据实际的Chr来修改。
    color = rdylgn-11-div-1
    </rule>

    <rule>
    condition = var(chr1) eq "Chr2"
    color = rdylgn-11-div-2
    </rule>

    <rule>
    condition = var(chr1) eq "Chr3"
    color = rdylgn-11-div-3
    </rule>

    <rule>
    condition = var(chr1) eq "Chr4"
    color = rdylgn-11-div-4
    </rule>

    <rule>
    condition = var(chr1) eq "Chr5"
    color = rdylgn-11-div-5
    </rule>

    <rule>
    condition = var(chr1) eq "Chr6"
    color = rdylgn-11-div-6
    </rule>

    <rule>
    condition = var(chr1) eq "Chr7"
    color = rdylgn-11-div-7
    </rule>

    <rule>
    condition = var(chr1) eq "Chr8"
    color = rdylgn-11-div-8
    </rule>

    <rule>
    condition = var(chr1) eq "Chr9"
    color = rdylgn-11-div-9
    </rule>

    <rule>
    condition = var(chr1) eq "Chr10"
    color = rdylgn-11-div-10
    </rule>

    <rule>
    condition = var(chr1) eq "Chr11"
    color = rdylgn-11-div-11
    </rule>

    <rule>
    condition = var(chr1) eq "Chr12"
    color = rdylbu-9-div-1
    </rule>

    <rule>
    condition = var(chr1) eq "Chr13"
    color = rdylbu-9-div-2
    </rule>

    <rule>
    condition = var(chr1) eq "Chr14"
    color = rdylbu-9-div-3
    </rule>

    <rule>
    condition = var(chr1) eq "Chr15"
    color = rdylbu-9-div-4
    </rule>

    <rule>
    condition = var(chr1) eq "Chr16"
    color = rdylbu-9-div-5
    </rule>

    <rule>
    condition = var(chr1) eq "Chr17"
    color = rdylbu-9-div-6
    </rule>
    </link>
    </links>
    <ideogram>
    <spacing>
    default = 0.005r

    #设置染色体之间空出个距离，用来标图例a,b,c,d,e,f
    <pairwise Chr17;Chr1>  #染色体名称是karyotype.Spo.txt中第3列的名称，根据你实际的物种第一条和最后一条来修改此处
    spacing = 5r #设置第一个和最后一个染色体中间空出5*default（0.005r）的距离
    </pairwise>

    </spacing>

    radius           = 0.85r #指定画图的半径（此半径和plots里的半径不一致，这个是总的0.85r，决定了整个圈图的最大的半径,plots里的0.99r半径是以这个0.85r作为100%来分配99%）
    thickness        = 10p #坐标轴的染色体的厚度
    fill             = yes
    show_bands      = yes #设置染色体条带填充，如果不想用条带填充，只用颜色，就设置为no
    fill_bands      = yes
    stroke_color     = dgrey
    stroke_thickness = 2p
    show_label     = yes #展示label
    label_font     = default # 字体
    label_radius   = dims(ideogram,radius) + 0.075r #位置
    label_size     = 12 # 字体大小
    label_parallel = yes # 是否平行
    label_format   = eval(sprintf("%s",var(chr))) # 格式
    </ideogram>

    ##以下部分通常不做修改
    #设置图片参数
    <image>
    #覆盖原来的图片半径参数
    #radius* = 500
    </image>
    #设置颜色，字体，填充模式的配置信息：
    #系统与debug参数：

#### 3.3.3运行
完成以上配置文件后运行命令
```
circos -conf circos.conf
```
即可  
如果是两个物种之间的比较的圈图，直接将对应的文件合并成一个文件即可，需要注意的是，两个物种的染色体名称需要不同


***<后话>***  
circos图可展示的远不止以上这些，还可以展示染色体可及性，表达水平，只要最后文件与上面所展示的文件格式类似就可以展示，不得不感慨，这真的是技术与美学的一场碰撞。


