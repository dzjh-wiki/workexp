# 胡牌算法 

## 关于胡牌算法

如何才算胡牌？  
  * 当手牌能够完整地组合刻子（3张一样的牌）或顺子，且有一个将对（2张一样的牌）时，这幅手牌满足胡牌的基本条件。

核心算法：  
  * 组合手牌算法：完整将手牌拆分成各个组合。


## 组合手牌算法

### 输入数据
&nbsp;数据样本(D)：  
  * 普通牌值数据
  * 癞子牌（可充当其他牌）值数据

&nbsp;组合器(G)：  
  * 将、刻、顺

### 输出结果
  * 手牌组合列表

### 算法数据
每次递归中，有用于缓存递归中查找组合结果的数据：  
  * (1)用于缓存找出组合的当前胡牌组合数据；
  * (2)用于缓存已用于参与组合的牌数据；

### 算法实现
一重循环，一个判断：  
  * (1)循环遍历传入的组合器，以未用于组合的数据为起点，查找组合；
  * (2)判断是否找到组合；若是则保存数据，并递归进入下一次组合；否则不处理。

### 算法分析
#### 重复组合
在使用递归穷举胡牌组合时，难免出现组合重复的问题，这时候，需要考虑如何去除重复组合的问题。  
  * 根据各个组合中牌值（牌面值），生成胡牌组合的唯一值。

#### 缺失组合
在使用递归穷举胡牌组合，尤其是有癞子牌（可充当其他牌的牌）时，有可能会出现找出的胡牌组合不全问题。  
  * 其一，需改变各个组合器的实现，力求穷举所有组合情况。然而这也意味着，会出现更多的重复组合，及大地影响性能。
  * 其二，可以从开始组合数据入手，从组合起始点来查找更多的组合情况。然而，对性能也存在影响。

#### 组合次数
设，组合器个数为G，手牌张数为D。
为了计算的方便，这里假设，每个组合器组合牌张数俱为M。  

##### 成功组合的最少次数
当每次递归时，唯最后一次才成功找出组合，则可知最后的组合次数为：  
`count = G * (D/M)`

##### 成功组合的最多次数
当每次递归都能找到组合，然后进入下一个递归，则可知最后的组合次数为：  
`count = G ^ (D/M)`

### 附录
#### 接口函数
```go
package hu

import (
	"bytes"
	"hu/combiner"
	"sort"
	"strconv"
)

type group []int

type huGroup [][]group

/* 通用胡牌算法
@params cards 传入手牌
@params lzCnt 可用癞子数量
@params combiners 组合器
@return huGroups 胡牌组合列表
*/
func GetHuGroups(cards []int, lzCnt int, combiners []combiner.Combiner) (huGroups []huGroup) {
	curHuGroup := make(huGroup, len(combiners))
	usedMap := make(map[int]bool)
	if len(cards) > 0 {
		checkGroupsByRecursion(0, cards, lzCnt, combiners, &huGroups, curHuGroup, usedMap)
	} else {
		checkGroupsByRecursion(-1, cards, lzCnt, combiners, &huGroups, curHuGroup, usedMap)
	}
	return uniqueHuGroups(huGroups)
}

// 递归检测胡牌组合
func checkGroupsByRecursion(startIdx int, cards []int, lzCnt int, combiners []combiner.Combiner, huGroups *[]huGroup, curHuGroup huGroup, usedMap map[int]bool) {
	for i := 0; i < len(combiners); i++ {
		if group := combiners[i](startIdx, cards, lzCnt, usedMap); group != nil {
			// 保存组合
			curHuGroup[i] = append(curHuGroup[i], group)
			for _, card := range group {
				if card == -1 {
					lzCnt--
				} else {
					usedMap[card] = true
				}
			}
			// 递归检测
			if len(usedMap) == len(cards)+lzCnt {
				*huGroups = append(*huGroups, copyHuGroup(curHuGroup))
			} else {
				nextIdx := getNextUnUsedIndex(startIdx, cards, usedMap)
				checkGroupsByRecursion(nextIdx, cards, lzCnt, combiners, huGroups, curHuGroup, usedMap)
			}
			// 还原组合
			curHuGroup[i] = curHuGroup[i][:len(curHuGroup[i])-1]
			for _, card := range group {
				if card == -1 {
					lzCnt++
				} else {
					delete(usedMap, card)
				}
			}
		}
	}
}

// 获取牌中下个还未使用的index
func getNextUnUsedIndex(startIdx int, cards []int, usedMap map[int]bool) int {
	for i := 0; i < len(cards); i++ {
		idx := (i + startIdx) % len(cards)
		if val, ok := usedMap[cards[idx]]; !ok || !val {
			return idx
		}
	}
	return -1
}

// 拷贝胡牌组合
func copyHuGroup(oriHuGroup huGroup) huGroup {
	newHuGroup := make(huGroup, len(oriHuGroup))
	for i := 0; i < len(oriHuGroup); i++ {
		newHuGroup[i] = make([]group, len(oriHuGroup[i]))
		for j := 0; j < len(oriHuGroup[i]); j++ {
			newHuGroup[i][j] = oriHuGroup[i][j]
		}
	}
	sortHuGroup(newHuGroup)
	return newHuGroup
}

/*
组合列表[用于对手牌的组合进行排序]
实现排序所要求的的Len、Less和Swap方法
*/
type groupList []group

func (this groupList) Len() int {
	return len(this)
}

func (this groupList) Less(i, j int) bool {
	return this[i][0] < this[j][0]
}

func (this groupList) Swap(i, j int) {
	this[i], this[j] = this[j], this[i]
}

// 排序胡牌组合
func sortHuGroup(oriHuGroup huGroup) {
	for i := 0; i < len(oriHuGroup); i++ {
		sort.Sort(groupList(oriHuGroup[i]))
	}
}

// 组合去重
func uniqueHuGroups(huGroups []huGroup) (newHuGroups []huGroup) {
	huGroupValMap := make(map[string]bool)
	var buffer bytes.Buffer
	for _, theHuGroup := range huGroups {
		buffer.Reset()
		for i := 0; i < len(theHuGroup); i++ {
			buffer.WriteString(strconv.Itoa(i)) // 组合类型
			buffer.WriteString("_")
			for j := 0; j < len(theHuGroup[i]); j++ {
				for k := 0; k < len(theHuGroup[i][j]); k++ {
					buffer.WriteString(strconv.Itoa(theHuGroup[i][j][k] >> 8)) // 取牌的byte值
					buffer.WriteString("_")
				}
			}
		}
		val := buffer.String()
		if _, isOk := huGroupValMap[val]; !isOk {
			newHuGroups = append(newHuGroups, theHuGroup)
			huGroupValMap[val] = true
		}
	}
	//fmt.Println("uniqueHuGroups huGroupValMap:", huGroupValMap)
	return newHuGroups
}


/* 特殊胡牌算法
@params cards 传入手牌
@params lzCnt 可用癞子数量
@params combiners 普通组合器
@params sCombiners 特殊组合器（递归中只会调用一次）
@return huGroups 胡牌组合列表
*/
func GetSpecialHuGroups(cards []int, lzCnt int, combiners, sCombiners []combiner.Combiner) (huGroups []huGroup) {
	curHuGroup := make(huGroup, len(combiners)+len(sCombiners))
	startIdx, usedMap, usedCbMap := 0, make(map[int]bool), make(map[int]bool)
	if len(cards) == 0 {
		startIdx = -1
	}
	checkSpecialHuGroups(startIdx, cards, lzCnt, combiners, sCombiners, &huGroups, curHuGroup, usedMap, usedCbMap)
	return uniqueHuGroups(huGroups)
}

// 递归检测胡牌组合
func checkSpecialHuGroups(startIdx int, cards []int, lzCnt int, combiners, sCombiners []combiner.Combiner, huGroups *[]huGroup, curHuGroup huGroup, usedMap, usedCbMap map[int]bool) {
	for i := 0; i < len(combiners)+len(sCombiners); i++ {
		// 获取组合
		var g group
		if i < len(combiners) {
			g = combiners[i](startIdx, cards, lzCnt, usedMap)
		} else {
			if _, ok := usedCbMap[i]; !ok { // 若未曾使用过，才使用该组合器进行组合
				g = sCombiners[i-len(combiners)](startIdx, cards, lzCnt, usedMap)
			}
		}
		if g == nil || len(g) == 0 {
			continue
		}
		// 标记使用过的组合器
		usedCbMap[i] = true
		// 保存组合
		curHuGroup[i] = append(curHuGroup[i], g)
		for _, card := range g {
			if card == -1 {
				lzCnt--
			} else {
				usedMap[card] = true
			}
		}
		// 递归检测
		if len(usedMap) == len(cards)+lzCnt {
			*huGroups = append(*huGroups, copyHuGroup(curHuGroup))
		} else {
			nextIdx := getNextUnUsedIndex(startIdx, cards, usedMap)
			checkSpecialHuGroups(nextIdx, cards, lzCnt, combiners, sCombiners, huGroups, curHuGroup, usedMap, usedCbMap)
		}
		// 还原组合
		curHuGroup[i] = curHuGroup[i][:len(curHuGroup[i])-1]
		for _, card := range g {
			if card == -1 {
				lzCnt++
			} else {
				delete(usedMap, card)
			}
		}
		// 还原标记使用过的组合器
		delete(usedCbMap, i)
	}
}


/* 优先组合算法
@params cards 传入手牌
@params lzCnt 可用癞子数量
@params combiners 普通组合器
@params sCombiners 特殊组合器（每次递归只会调用一次）
@return huGroups 胡牌组合列表
*/
func GetPriorityGroups(cards []int, lzCnt int, combiners []combiner.Combiner) (huGroups []huGroup) {
	if len(cards) > 0 {
		for i := 0; i < len(cards); i++ {
			if hg := checkPriorityGroups(i, cards, lzCnt, combiners); hg != nil {
				huGroups = append(huGroups, hg)
			}
		}
	} else if hg := checkPriorityGroups(-1, cards, lzCnt, combiners); hg != nil {
		huGroups = append(huGroups, hg)
	}
	return uniqueHuGroups(huGroups)
}

func checkPriorityGroups(startIdx int, cards []int, lzCnt int, combiners []combiner.Combiner) huGroup {
	usedMap, curHuGroup, isCheckout := make(map[int]bool), make(huGroup, len(combiners)), false
	for len(usedMap) < len(cards)+lzCnt {
		isCheckout = false // 重置检出标记
		for i := 0; i < len(combiners); i++ {
			if g := combiners[i](startIdx, cards, lzCnt, usedMap); g != nil {
				// 保存组合
				curHuGroup[i] = append(curHuGroup[i], g)
				for _, card := range g {
					if card == -1 {
						lzCnt--
					} else {
						usedMap[card] = true
					}
				}
				// 重置检测七点
				startIdx = getNextUnUsedIndex(startIdx, cards, usedMap)
				isCheckout = true // 标记检出
				break
			}
		}
		if !isCheckout { // 未检测出组合，直接返回，避免死循环
			return nil
		}
	}
	return curHuGroup
}

```

#### 组合器（函数）
```go
package combiner

type Combiner func(startIdx int, cards []int, lzCards []int, usedMap map[int]bool) []int

// 组合将
func CombineJiang(startIdx int, cards []int, lzCnt int, usedMap map[int]bool) []int {
	cardCount := 2
	if len(cards)+lzCnt < cardCount {
		return nil
	}
	curGroup := make([]int, cardCount)
	if startIdx >= 0 {
		// 查找普通牌
		startCard := cards[startIdx]
		curGroup[0] = startCard
		for i := 1; i < len(cards); i++ {
			card := cards[(i+startIdx)%len(cards)]
			if val, ok := usedMap[card]; !ok || !val {
				if curGroup[1] == 0 && (card>>8 == startCard>>8) {
					curGroup[1] = card
					return curGroup
				}
			}
		}
	}
	// 查找癞子牌
	for i := 0; i < lzCnt; i++ {
		if curGroup[0] == 0 {
			curGroup[0] = -1
		} else if curGroup[1] == 0 {
			curGroup[1] = -1
			return curGroup
		}
	}
	return nil
}

// 组合刻子
func CombineKe(startIdx int, cards []int, lzCnt int, usedMap map[int]bool) []int {
	cardCount := 3
	if len(cards)+lzCnt < cardCount {
		return nil
	}
	curGroup := make([]int, cardCount)
	if startIdx >= 0 {
		// 查找普通牌
		startCard := cards[startIdx]
		curGroup[0] = startCard
		for i := 1; i < len(cards); i++ {
			card := cards[(i+startIdx)%len(cards)]
			if val, ok := usedMap[card]; !ok || !val {
				if card>>8 == startCard>>8 {
					if curGroup[1] == 0 {
						curGroup[1] = card
					} else {
						curGroup[2] = card
						return curGroup
					}
				}
			}
		}
	}
	// 查找癞子牌
	for i := 0; i < lzCnt; i++ {
		if curGroup[0] == 0 {
			curGroup[0] = -1
		} else if curGroup[1] == 0 {
			curGroup[1] = -1
		} else {
			curGroup[2] = -1
			return curGroup
		}
	}
	return nil
}

// 组合顺子
func CombineShun(startIdx int, cards []int, lzCnt int, usedMap map[int]bool) []int {
	cardCount := 3
	if len(cards)+lzCnt < cardCount {
		return nil
	}
	curGroup, setTime := make([]int, cardCount), 0
	setGroup := func(idx, card int) {
		curGroup[idx] = card
		setTime++
	}
	if startIdx >= 0 {
		// 查找普通牌
		startCard := cards[startIdx]
		setGroup(0, startCard)
		for i := 1; i < len(cards); i++ {
			card := cards[(i+startIdx)%len(cards)]
			if val, ok := usedMap[card]; !ok || !val {
				if curGroup[1] == 0 && (card>>8-startCard>>8 == 1) {
					setGroup(1, card)
				} else if curGroup[2] == 0 && (card>>8-startCard>>8 == 2) {
					setGroup(2, card)
				}
				if setTime == 3 {
					return curGroup
				}
			}
		}
	}
	// 查找癞子牌
	for i := 0; i < lzCnt; i++ {
		if curGroup[0] == 0 {
			setGroup(0, -1)
		} else if curGroup[1] == 0 {
			setGroup(1, -1)
		} else if curGroup[2] == 0 {
			setGroup(2, -1)
		}
		if setTime == 3 {
			return curGroup
		}
	}
	return nil
}

// 组合边顺
func CombineBianShun(startIdx int, cards []int, lzCnt int, usedMap map[int]bool) []int {
	cardCount := 2
	if len(cards)+lzCnt < cardCount {
		return nil
	}
	curGroup := make([]int, cardCount)
	if startIdx >= 0 {
		// 查找普通牌
		startCard := cards[startIdx]
		curGroup[0] = startCard
		for i := 1; i < len(cards); i++ {
			card := cards[(i+startIdx)%len(cards)]
			if val, ok := usedMap[card]; !ok || !val {
				if curGroup[1] == 0 && (card>>8-startCard>>8 == 1) {
					curGroup[1] = card
					return curGroup
				}
			}
		}
	}
	// 查找癞子牌
	for i := 0; i < lzCnt; i++ {
		if curGroup[0] == 0 {
			curGroup[0] = -1
		} else if curGroup[1] == 0 {
			curGroup[1] = -1
			return curGroup
		}
	}
	return nil
}

// 组合夹顺
func CombineJiaShun(startIdx int, cards []int, lzCnt int, usedMap map[int]bool) []int {
	cardCount := 2
	if len(cards)+lzCnt < cardCount {
		return nil
	}
	curGroup := make([]int, cardCount)
	if startIdx >= 0 {
		// 查找普通牌
		startCard := cards[startIdx]
		curGroup[0] = startCard
		for i := 1; i < len(cards); i++ {
			card := cards[(i+startIdx)%len(cards)]
			if val, ok := usedMap[card]; !ok || !val {
				if curGroup[1] == 0 && (card>>8-startCard>>8 == 2) {
					curGroup[1] = card
					return curGroup
				}
			}
		}
	}
	// 查找癞子牌
	for i := 0; i < lzCnt; i++ {
		if curGroup[0] == 0 {
			curGroup[0] = -1
		} else if curGroup[1] == 0 {
			curGroup[1] = -1
			return curGroup
		}
	}
	return nil
}

// 组合单牌
func CombineDan(startIdx int, cards []int, lzCnt int, usedMap map[int]bool) []int {
	cardCount := 1
	if len(cards)+lzCnt < cardCount {
		return nil
	}
	curGroup := make([]int, cardCount)
	if startIdx >= 0 {
		// 查找普通牌
		startCard := cards[startIdx]
		curGroup[0] = startCard
		return curGroup
	}
	// 查找癞子牌
	for i := 0; i < lzCnt; i++ {
		if curGroup[0] == 0 {
			curGroup[0] = -1
			return curGroup
		}
	}
	return nil
}
```