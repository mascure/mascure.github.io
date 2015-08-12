---
layout: post
title:  "eight queen"
date:   2015-08-07 13:17:58
categories: jekyll update
---
代码写的越简洁越容易维护，不能盲目地用空间换时间，在可行的情况下应该用尽可能少的变量。这个问题就没有必要再开两个数组保存对角线方向的占用情况，因为维护这两个数组要花费更多地代码，而且排错很不容易。

回溯和递归两种方法。
{% highlight c++%}
int a[100],cnt=0;
int n=8;
//检查第n个和前n-1个是否没有冲突
bool check(int a[],int n){
    for (int i=1; i<=n-1; ++i) {
        if (abs(a[n]-a[i])==n-i||a[i]==a[n]) {
            return false;
        }
    }
    return true;
}

void queen(int N){
    int k=1;
    a[1]=0;
    cnt=0;
    while (k>0) {
        //当前为第k行，放置到下一个
        a[k]=a[k]+1;
        //依次查找第k行可以放置的位置
        while (a[k]<=N&&!check(a,k)) {
            a[k]++;
        }
        //找到一个可行的位置
        if (a[k]<=N) {
            //到达最后一行，说明得到一个解
            if (k==N) {
                cout<<++cnt<<":"<<endl;
                for (int i=1; i<=N; ++i) {
                    cout<<i<<","<<a[i]<<endl;
                }
            }
            //继续放下一行
            else{
                k=k+1;
                a[k]=0;
            }
        }
        //没有找到可行的位置，回溯到上一行
        else{
            k--;
        }
    }
}
void queen2(int k){
    //得到一个解
    if (k==n+1) {
        cout<<++cnt<<":"<<endl;
        for (int i=1; i<=n; ++i) {
            cout<<i<<","<<a[i]<<endl;
        }
        return;
    }
    //依次尝试放第1-n的位置
    for (int i=1; i<=n; ++i) {
        a[k]=i;
        if (check(a, k)) {
            //放下一行
            queen2(k+1);
        }
    }
}
int main()
{
    queen2(1);
    return 0;
}
{% endhighlight %}

