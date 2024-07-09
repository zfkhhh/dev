1. 两数和
https://leetcode.cn/problems/two-sum/solutions/434597/liang-shu-zhi-he-by-leetcode-solution/
`
func twoSum(nums []int, target int) []int {
    numMap := make(map[int]int)
    for i, v := range nums {
       if index, exist := numMap[target-v]; exist {
         return  []int{i, index}
       }
       numMap[v] = i
    }
    return []int{}
}
`

2. 最长连续序列
https://leetcode.cn/problems/longest-consecutive-sequence/description/?envType=study-plan-v2&envId=top-100-liked

变量存放所有的数，遍历每一个数，找当前数为开头的序列
`
func longestConsecutive(nums []int) (res int) {
    numMap := make(map[int]bool)
    for _, v := range nums {
        numMap[v] = true
    }
    
    for num,_ := range numMap {
        var temp int
        // 当前数字是开头
        if !numMap[num-1] {
            var i int
            for numMap[num+i] {
                temp++
                i++
            }
        }
        res = slices.Max([]int{temp, res})
    }
    return 
}
`