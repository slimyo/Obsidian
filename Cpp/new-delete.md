## 函数内new，函数外释放

- 法1
```cpp
int*LPS(const string pattern){
            // Longest prefix suffix算法
            int* lps = new int[pattern.size()]{0};
            int j =0;// j表示前缀长度（当前最大前缀最后一个字符位置）
            //从1开始（0位置最长前后缀一定为0）
            for(int i=1;i<pattern.size();i++){
                // i表示
                while(j>0&&pattern[i]!=pattern[j]){
                    j = lps[j-1];
                }
                if(pattern[i]==pattern[j]){
                    j++;
                }
                lps[i] = j;
            }
            return lps;
        }
...
void main(){
int*lps = LPS(s);
...
delete[]lps;
}
```