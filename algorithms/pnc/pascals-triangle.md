# Pascal's Triangle

```cpp
vector<vector<int>>pascalTriangle(int n){

    vector<vector<int>>tri(n+1);

    for(int i=0;i<=n;i++){
        tri[i] = vector<int>(i+1,1);
        for(int j=1;j<i;j++){
            tri[i][j] = tri[i-1][j]+tri[i-1][j-1];
        }
    }

    return tri;
}
```
