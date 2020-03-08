class Solution {

 public int climbStairs(int n) {

​	int count =0;	

​	int j =0;

​	for(int i=n;i>=0;i =i-2,j++;){

​		count = count + getResult(i,j);

​	}

​	return count;

  }

getResult(x1,x2){

int i=x1;int j= x2;

​	int result =1;

​	if(i<j){

​		int k =j;

​		j=i;

​		i=k;

​	}

​	for(int x =i+j; x>j;x--){

​		result = result*x;

​	}

​	for(int x = j; x >0;x--){

​		result = result/x

​	}

​	return result;

}













