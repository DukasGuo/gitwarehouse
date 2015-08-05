#Anaconda  中文速查表
作者：迪卡斯
提示，速查表中没有的，请用下面代码来查：
> conda create -- help

###管理conda 和 anaconda

<table>
    <tr>
        <td>conda info  </td> <td>验证conda的安装 检查版本</td>
      
    </tr>
    <tr>  <td>conda updata conda</td> <td> 更新conda和环境管理器到最新版本
    </tr>
     <tr>  <td>conda updata anaconda</td> <td>更新anaconda 元包</td> 
     </tr>     
</table>

###管理环境
<table>
	<tr>
	<td>conda info --envs 或者 <br>
	 conda info -e</td><td>显示环境列表，活跃环境用*标明</td>
	</tr>
	<tr>
	<td>conda create   --name snowflakes  biopython    或者<br> <br>  conda create -n snowflakes biopython </td><td>创建一个环境并安装相应程序<br> <strong>提示:</strong>为避免依赖冲突，conda将在同一时间安装所有相应的程序<br><strong>提示:</strong> 环境自动安装在conda索引下的envs 索引中。你可以自己指定一个不同的路径</td>
	</tr>
	<tr>
	<td>source activate snowflakes (Linux, Mac)<br>activate snowflakes(Windows)</td><td>激活新环境</td>
	</tr>
	<tr>
	<td>conda create -n bunnies python=2.7 astroid</td><td>创建环境并指定python版本</td>
	</tr>
	<tr>
	<td>conda create -n flowers --clone snowflakes</td><td>精确复制环境</td>
	</tr>
</table>
 
###管理Python
<table>
	<tr>
	<td>conda search --full-name python <br>conda search -f python
</td><td>检查可用的Python版本并安装</td>
	</tr>
	<tr>
	<td>conda create -n snakes python=2.7
	</td><td>给新环境安装不同版本的Python</td>
	</tr>
	<tr>
	<td>source activate snakes (Linux, Mac) <br>activate snakes (Windows)</td><td>切换到有不同Python版本的新环境</td>
	</tr>
</table>

###管理 .condarc 配置
<table>
	<tr>
	<td>conda config --get</td><td>从 .condarc文件中得到所有的键和值</td>
	</tr>
	<tr>
	<td>conda config --get channels</td><td>从 .condarc文件中获得从键到值的通道</td>
	</tr>
	<tr>
	<td>conda config --add channels pandas</td><td>在conda查找到包的位置 通过通道添加一个新值</td>
</table>
###管理Python包
<table>
	<tr>
	<td>conda list</td><td>预览活跃环境中的包及其版本</td>
	</tr>
	<tr>
	<td>conda search PackageName</td><td>寻找一个包，确定它是否可被conda安装</td>
	</tr>
	<tr>
	<td>conda install PackageName </td><td>当前环境安装一个新包<br><strong>提示:</strong> conda可安装包列表。在这里http://docs.continuum.io/anaconda/pkg-docs
	 </td>
	</tr>
	<tr>
	<td>conda update PackageName</td><td>更新一个包</td>
	</tr>
	<tr>
	<td>conda search --override-channels -c pandas bottleneck</td><td>在特定位置查找一个包（Anaconda.org 的pandas频道）</td>
	</tr>
	<tr>
	<td>conda install -c pandas bottleneck</td><td>从一个特定频道安装包</td>
	</tr>
	<tr>
	<td>conda search --override-channels -c defaults PackageName</td><td>在Anaconda 库中查到一个包</td>
	</tr>
	<tr>
	<td>source activate bunnies (Linux, Mac)<br>
	activate bunnies (Windows)<br>
	pip install see</td><td>在激活环境中用 pip安装包</td>
	</tr>
	<tr>
	<td>conda install iopro accelerate</td><td>安装商业系列 的包</td>
	</tr>
</table>
###删除包或环境
<table>
	<tr>
	<td>conda remove --name bunnies PackageName</td><td>从任意位置删除包</td>
	</tr>
	<tr>
	<td>conda remove PackageName</td><td>从活跃位置删除包</td>
	</tr>
	<tr>
	<td>conda remove --name bunnies PackageName1 PackageName2</td><td>从任意位置删除多个包。</td>
	</tr>
	<tr>
	<td>conda remove --name snakes --all</td><td>删除一个环境</td>
	</tr>
	</table>

###后记： 
conda支持的包 python2.7版本的有340个，python3.4版本的有279个。数量上并不如pip，量化投资用的 tushare ，PyAlgoTrade都不支持。当conda不支持时，conda的包管理指令和包删除指令就不顶用了。而使用pip管理包时注意conda环境