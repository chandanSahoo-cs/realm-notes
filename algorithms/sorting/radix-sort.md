# Radix Sort

```cpp
    #define ll long long
    int n = nums.size();

    int mx = *max_element(nums.begin(),nums.end());
    int k = 0;

    while(mx){
        k++;
        mx/=10;
    }
    
    ll div = 1;

    vector<int>a = nums;

    while(k--){
        vector<vector<int>>bin(10);

        for(auto &ele:a){
            bin[(ele/div)%10].push_back(ele);
        }

        vector<int>b;

        for(int i=0;i<10;i++){
            for(auto ele:bin[i]){
                b.push_back(ele);
            }
        }
        a = b;
        div*=10;
    }
```