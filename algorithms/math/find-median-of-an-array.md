# Find median of an array

```cpp

int main(){
    vector<int>nums;
    ll n = nums.size();

    vector<ll>cost(n);

    int i = 0, j = 0;
    multiset<int>left,right;
    ll leftSm = 0, subSm=0;

    auto rebalance = [&](){
        if(left.size()>0 && right.size()>0 && *left.rbegin()>*right.begin()){
            int a = *left.rbegin();
            int b = *right.begin();

            leftSm+=b-a;

            left.erase(left.find(a));
            right.erase(right.find(b));

            left.insert(b);
            right.insert(a);
        }
    };

    while(j<n){
        if(left.size()<=right.size()){
            left.insert(nums[j]);
            leftSm+=nums[j];
        }else right.insert(nums[j]);

        subSm+=nums[j];

        rebalance();

        if(j-i+1==x){
            int med = *left.rbegin(); // this is the median
            cost[i] = left.size()*med-leftSm+(subSm-leftSm)-right.size()*med;

            if(left.find(nums[i])!=left.end()){
                left.erase(left.find(nums[i]));
                leftSm-=nums[i];
            }else right.erase(right.find(nums[i]));

            subSm-=nums[i];
            i++;

            rebalance();
        }
        j++;
    }

    return 0;
}
```
