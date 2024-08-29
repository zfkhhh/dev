import "container/list"

type LRUCache struct {
	keyMap   map[int]*list.Element
	list     *list.List
	capacity int
}

type entry struct {
	key, value int
}

func Constructor(capacity int) LRUCache {
	return LRUCache{
		keyMap:   make(map[int]*list.Element),
		list:     list.New(),
		capacity: capacity,
	}
}

func (this *LRUCache) Get(key int) int {
	if node, exist := this.keyMap[key]; exist {
		this.list.MoveToFront(node)
		return node.Value.(entry).value
	}
	return -1
}

func (this *LRUCache) Put(key int, value int) {
	if node, exist := this.keyMap[key]; exist {
		this.list.MoveToFront(node)
		node.Value = entry{key, value}
		return
	}
	node := entry{key, value}
	this.keyMap[key] = this.list.PushFront(node)
	if len(this.keyMap) > this.capacity {
		delete(this.keyMap, this.list.Remove(this.list.Back()).(entry).key)
	}
}