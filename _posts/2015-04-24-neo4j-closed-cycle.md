---
title: 使用neo4j进行交易闭环分析
layout: post
category: database
tagline: database
tags: [neo4j, graphdb]
---

目前手头有一份一些公司的发票数据，反应了这些公司之间的交易情况。本来主要目的是做一些fraud analysis，看看能不能找出一些虚假/欺诈的交易，进而分析公司的账务情况。

刚好最近在看neo4j，想试一下graph analysis，数据提供者也觉得分析一下不同公司之间交易闭环也蛮有趣的。试了一下，用neo4j和python的组合来完成这事儿还是非常容易的。

#问题定义与分析
数据是不同商家(我称为dealer)之间的交易(称为transaction)，经过数据清洗和简化，每条数据的格式可以认为是这样：

    dealerA, productA, dealerB, amount

表示dealerA将数额为amount的productA卖给了dealerB。数据集中每个dealer即可能是卖家，也可能是买家。要解决的问题是，给定一定量这样的数据，发现交易闭环，即从一个dealer出发，通过不同的交易和其他dealer，最后还有人将产品卖回给最开始的dealer。

可以把这个问题抽象成一个有向图，dealer是图的节点，transaction是图的边，从卖家指向买家，权重为交易额。要解决的问题即是，给定一个图G(V,E)，找出所有的闭合子图。

#neo4j实现
这个问题可以全程使用neo4j实现，不过方便起见，还是用了python做胶水。

1. 首先，根据输入数据构造交易图谱，得到一个neo4j的数据库。
2. 第二步，使用neo4j的查询语言Cypher进行封闭子图查询。
3. 最后判断重复结果，输出。

构造图的过程中要注意的问题就是不能重复创建节点。Cypher中得merge/create unique语法其实不是特别方便，两者的语义都不是我想要实现的“创建关系r(x<sub>1</sub>, x<sub>2</sub>)及节点x<sub>1</sub>,x<sub>2</sub>，如果x<sub>1</sub>/x<sub>2</sub>存在则不重复创建”，只好分别对x<sub>1</sub>，x<sub>2</sub>和r进行merge操作。

构造的代码如下，商家定义为`Dealer`类，交易关系命名为`s2`，函数的输入`connection`为一个本地的neo4j数据库连接；输入文件格式如前述，第一列和第三列为卖方和买房的id。


	def create_graph(inpath, connection):

	    with codecs.open(inpath, encoding='utf-8') as f:
	        cursor = connection.cursor()
	        for line in f:
	            entries = line.split('\t')
	            a, b = entries[0].strip(), entries[2].strip()
	            cursor.execute("MERGE (x:Dealer {id:{0}}) "
	                           "MERGE (y:Dealer {id:{1}}) "
	                           "MERGE (x) -[:s2]-> (y)", a, b)
	            connection.commit()


最终输出的时候要判断找到的子图是不是重复的，由于起点不同，所以查询结果可能有重复。在这里我定义了一个闭合子图类，定义了它的哈希方法和判断相等的方法。还有一个需要注意的地方是，设定一个节点做起始节点，将找到的两个子图结果拼接起来仍是一个合法的结果，例如找到两个结果`a--b--a`和`a--c--a`，拼接起来的结果`a--b--a--c--a`仍是一个合法的图查询结果，但是不是我们想要的，所以还需要过滤掉这种倒8字型的子图。这个`Circle`类的代码如下。

	class Circle(object):

	    def __init__(self, nodes):

	        self.valid = True
	        if nodes.count(nodes[0]) > 2: self.valid = False
	        self.nodes = nodes[1:]

	    def __hash__(self):

	        return (frozenset(self.nodes).__hash__(), len(self.nodes)).__hash__()

	    def __eq__(self, other):

	        return self.__hash__() == other.__hash__() if isinstance(other, Circle) else False

	    def toString(self):

	        return '->'.join(self.nodes)

查询并将结果写入文件：

	def search4circle(outpath, connection):

	    cursor = connection.cursor()
	    circles = []
	    with codecs.open(outpath, 'w', 'utf-8') as fo:
	        for p in cursor.execute("START n=node(*) "
	                                "MATCH p=n-[:s2*2..10]->n "
	                                "RETURN nodes(p)"):
	            circles.append(map(lambda x: x.get('id'), p[0]))
	        fo.write('\n'.join(map(lambda x: x.toString(),
	                               filter(lambda x: x.valid,
	                                      set(map(lambda x: Circle(x), circles))))))
	    connection.commit()

我指定的子图边的数量是2-10个，交易系统更复杂、闭环长度可能更长的时候应该适当增加上界。我的数据只是toy级别的，几万条交易数据而已，跑下来不到一分钟，不算很短了。具体原因我没有深究，我猜测是查询之前建立索引比较耽误工夫，因为不同边长度的结果几乎是一起出来的。如果是建索引比较花时间，那就不太会影响线上使用的速度。

总体感觉上，Cypher作为查询语言不算难用，相较于SPARQL，更类似SQL。性能上，neo4j是跑在jvm上的，应该优化不算深，也有不少人诟病neo4j的速度，不过可能是我的数据量小，性能瓶颈不是很明显，至少不比Jena之类的框架差。

一段时间内应该不会做太多知识图谱相关的工作了，所以之前更深入研究graphdb的计划可能就要放慢节奏了，更多地会在个人兴趣项目上对相关内容进行一些尝试。
