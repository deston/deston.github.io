---
layout: post
title: 关于Activity和Fragment销毁重建的一些问题
---


众所周知，Activity提供了onSaveInstanceState和onRestoreInstanceState方法，用于当activity被系统销毁时保存一些数据，首先我们来看看他们被触发的时机。  
onSaveInstanceState和onRetsoreInstanceState方法名字看起来很像，给人的感觉就是一对好基友，结对出现。可惜不是这样。  
看看android文档中对onSaveInstanceState的解释：  
>This method is called before an activity may be killed so that when it comes back some time in the future it can restore its state. For example, if activity B is launched in front of activity A, and at some point activity A is killed to reclaim resources, activity A will have a chance to save the current state of its user interface via this method so that when the user returns to activity A, the state of the user interface can be restored via onCreate(Bundle) or onRestoreInstanceState(Bundle).             

就是说一个activity“有可能”被系统回收时，onSaveInstanceState方法就会被调用。一般来说有以下场景可能会被系统回收掉：  

- 当用户按下HOME键时，activity进入后台。  
- 长按home键运行其他应用时。  
- 当用户按下电源键关闭屏幕时。  
- 从当前activity启动另外一个activity时，触发前一个activity的onSaveInstanceState方法。  
- 横竖屏切换时。  


而onRestoreInstanceState方法就没那么容易被触发。照例先看看官方文档的描述：  
> this method is called after onStart() when the activity is being re-initialized from a previously saved state, given here in savedInstanceState.   

可以看到该方法的触发时机，当activity确实被系统销毁了并且重建的时候，在onstart之后会被调用。  
总而言之，onSaveInstanceState方法很容易被触发，通常情况是activity进入了后台，开发者能够在此保存一些数据，而onRestoreInstanceState就没那么容易被触发了，必须是activity被销毁重建了之后才会触发。  
理解了activity关于这两个方法的调用时机，我们来看看fragment下这两个方法的触发时机。  
照例先看看谷歌官方文档中对于Fragment的onSaveInstanceState的描述:
> Called to ask the fragment to save its current dynamic state, so it can later be reconstructed in a new instance of its process is restarted. If a new instance of the fragment later needs to be created, the data you place in the Bundle here will be available in the Bundle given to onCreate(Bundle), onCreateView(LayoutInflater, ViewGroup, Bundle), and onActivityCreated(Bundle).  
> 
This corresponds to Activity.onSaveInstanceState(Bundle) and most of the discussion there applies here as well. Note however: this method may be called at any time before onDestroy(). There are many situations where a fragment may be mostly torn down (such as when placed on the back stack with no UI showing), but its state will not be saved until its owning activity actually needs to save its state.


简单来说就是和activity保持一致。并且fragment是没有onRestoreInstanceState方法的，保存的数据在fragment重建时从onCreate, onCreateView或者onActivityCreated中获取。  


我们知道fragment和acitivity的生命周期是相似的，并且fragment的生命周期是跟着activity一起走的，我们可以测试一下。 
写个activity和fragment，并把fragment添加到acitivity中的测试程序，acitivity的代码如下：

	public class MyActivity extends Activity {
	    public static final String TAG = "MyActivity";
	    private FrameLayout mFragmentContainer;

	    /**
	     * Called when the activity is first created.
	     */
	    @Override
	    public void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        Log.i(TAG, "-----------onCreate-----------");
	        setContentView(R.layout.main);
	        mFragmentContainer = (FrameLayout) findViewById(R.id.container);
	        Fragment myFragment = new MyFragment();
	        FragmentManager fragmentManager = getFragmentManager();
	        FragmentTransaction ft = fragmentManager.beginTransaction();
	        ft.add(R.id.container, myFragment, MyFragment.TAG);
	        ft.commitAllowingStateLoss();
	    }

	    @Override
	    protected void onStart() {
	        super.onStart();
	        Log.i(TAG, "-----------onStart-----------");
	    }

	    @Override
	    protected void onRestart() {
	        super.onRestart();
	        Log.i(TAG, "-----------onRestart-----------");
	    }

	    @Override
	    protected void onResume() {
	        super.onResume();
	        Log.i(TAG, "-----------onResume-----------");
	    }

	    @Override
	    protected void onRestoreInstanceState(Bundle savedInstanceState) {
	        super.onRestoreInstanceState(savedInstanceState);
	        Log.i(TAG, "-----------onRestoreInstanceState-----------");
	    }

	    @Override
	    protected void onSaveInstanceState(Bundle outState) {
	        super.onSaveInstanceState(outState);
	        Log.i(TAG, "-----------onSaveInstanceState-----------");
	    }

	    @Override
	    protected void onPause() {
	        super.onPause();
	        Log.i(TAG, "-----------onPause-----------");
	    }

	    @Override
	    protected void onStop() {
	        super.onStop();
	        Log.i(TAG, "-----------onStop-----------");
	    }

	    @Override
	    protected void onDestroy() {
	        super.onDestroy();
	        Log.i(TAG, "-----------onDestroy-----------");
	    } 
	}

在Activity创建的时候将Fragment添加到activity布局中的framelayout中，布局省略。  

 
再来看看fragment的代码

	public class MyFragment extends Fragment{
	    public static final String TAG = "MyFragment";
	
	    @Override
	    public void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        Log.i(TAG, "-----------onCreate-----------");
	    }
	
	    @Override
	    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
	        Log.i(TAG, "-----------onCreateView-----------");
	        View view = inflater.inflate(R.layout.fragment, null);
	        return view;
	
	    }
	
	    @Override
	    public void onPause() {
	        super.onPause();
	        Log.i(TAG, "-----------onPause-----------");
	    }
	
	    @Override
	    public void onStop() {
	        super.onStop();
	        Log.i(TAG, "-----------onStop-----------");
	    }
	
	    @Override
	    public void onSaveInstanceState(Bundle outState) {
	        super.onSaveInstanceState(outState);
	        Log.i(TAG, "-----------onSaveInstanceState-----------");
	    }
	
	    @Override
	    public void onResume() {
	        super.onResume();
	        Log.i(TAG, "-----------onResume-----------");
	    }
	
	    @Override
	    public void onStart() {
	        super.onStart();
	        Log.i(TAG, "-----------onStart-----------");
	    }
	
	    @Override
	    public void onAttach(Activity activity) {
	        super.onAttach(activity);
	        Log.i(TAG, "-----------onAttach-----------");
	    }
	
	    @Override
	    public void onDestroy() {
	        super.onDestroy();
	        Log.i(TAG, "-----------onDestroy-----------");
	    }
	
	    @Override
	    public void onDetach() {
	        super.onDetach();
	        Log.i(TAG, "-----------onDetach-----------");
	    }
	
	    @Override
	    public void onDestroyView() {
	        super.onDestroyView();
	        Log.i(TAG, "-----------onDestroyView-----------");
	    }
	}

只是简单的打印了一下log，接下来我们看看activity和fragment销毁重建的时候会打印出什么样的log，我们使用genymotion模拟器的横竖屏切换来模拟这个功能（也可以使用开发者选项的不保留活动来进行测试）。

在activity创建的时候的log如下：

	04-17 13:02:51.817      930-930/? I/MyActivity﹕ -----------onCreate-----------
	04-17 13:02:51.829      930-930/? I/MyFragment﹕ -----------onAttach-----------
	04-17 13:02:51.829      930-930/? I/MyFragment﹕ -----------onCreate-----------
	04-17 13:02:51.829      930-930/? I/MyFragment﹕ -----------onCreateView-----------
	04-17 13:02:51.841      930-930/? I/MyActivity﹕ -----------onStart-----------
	04-17 13:02:51.841      930-930/? I/MyFragment﹕ -----------onStart-----------
	04-17 13:02:51.841      930-930/? I/MyActivity﹕ -----------onResume-----------
	04-17 13:02:51.841      930-930/? I/MyFragment﹕ -----------onResume-----------

没什么异常现象，可以看到fragment的生命周期是跟着activity走的。接下来我们由竖屏切换到横屏看看

	04-17 13:04:52.613      930-930/com.example.Test I/MyFragment﹕ -----------onPause-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyActivity﹕ -----------onPause-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyFragment﹕ -----------onSaveInstanceState-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyActivity﹕ -----------onSaveInstanceState-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyFragment﹕ -----------onStop-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyActivity﹕ -----------onStop-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyFragment﹕ -----------onDestroyView-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyFragment﹕ -----------onDestroy-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyFragment﹕ -----------onDetach-----------
	04-17 13:04:52.613      930-930/com.example.Test I/MyActivity﹕ -----------onDestroy-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onAttach-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onCreate-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyActivity﹕ -----------onCreate-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onCreateView-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onAttach-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onCreate-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onCreateView-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyActivity﹕ -----------onStart-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onStart-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onStart-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyActivity﹕ -----------onRestoreInstanceState-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyActivity﹕ -----------onResume-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onResume-----------
	04-17 13:04:52.621      930-930/com.example.Test I/MyFragment﹕ -----------onResume-----------

由竖屏切换到横屏的生命周期变化，可以看到fragment的生命周期依然是随着activity一起的。可是我们当仔细看的时候，会发现一些奇怪的现象，为什么fragment在onDetch，然后重建时会打印出两个onAttach，两个onCreate，两个onStart，两个onResume，这是怎么一回事？  

有两种可能性：  

- 创建了两个fragment
- 同一个fragment走了两次生命周期


我们看看第一种可能性，创建了2个fragment，为了验证是不是这样，需要用到DDMS里的一个工具，hierarchy viewer。可以看到布局层次，结合fragment的布局文件和里面的控件来看，确实有两个同样的fragment。这里就暂时不贴图了，验证一下即可知道。  第二种可能性也就不攻自破了。

那么我们来分析一下为什么会创建两个fragment，fragment提供了一个onSaveInstanceState方法，用来给开发者保存数据的，既然如此我们很容易可以推测出来，有保存就有恢复的时候，从onSaveInstanceState方法的官方文档解释中我们知道保存的数据可以在fragment的onCreate,onCreateView 和onActivityCreated的时候进行恢复，那么很显然fragment和activity一样，在系统由于内存吃紧等原因销毁掉之后，再次进入会自动帮你重建。因此我们得出有一个fragment重建是很正常的事情。

那么另外一个fragment从哪里来？ activity销毁重建了，会重新走一遍生命周期，包括了onCreate方法，我们看看onCreate方法里做了什么。  
	
	 public void onCreate(Bundle savedInstanceState) {
		        super.onCreate(savedInstanceState);
		        Log.i(TAG, "-----------onCreate-----------");
		        setContentView(R.layout.main);
		        mFragmentContainer = (FrameLayout) findViewById(R.id.container);
		        Fragment myFragment = new MyFragment();
		        FragmentManager fragmentManager = getFragmentManager();
		        FragmentTransaction ft = fragmentManager.beginTransaction();
		        ft.add(R.id.container, myFragment, MyFragment.TAG);
		        ft.commitAllowingStateLoss();
		    }

创建了一个fragment并添加到布局中。  
答案来了，另外一个多余fragment是activity重建走onCreate时，我们自己加进去的。

总结一下，activity销毁重建，有两个fragment被创建，一个是系统自动创建的，另外一个是我们自己创建的。

分析一下销毁重建的log的打印顺序，很容易知道activity和fragment销毁重建时，系统先创建fragment，后创建acitivity。  并且我们自己添加进去的多余的fragment是后创建的。

那么我们怎么避免这样一个问题。现在谷歌很推崇fragment，如果是在activity的生命周期中创建fragment时，很容易就出现这样的问题。  

想了一些办法：  

- 在onSaveInstanceState的时候存入一个值，在销毁重建的时候看有没有这个值，有这个值就不再手动添加多余的fragment，使用系统创建的fragment。
- fragment的展示办法使用replace，而不是add。这样会保证只有一个fragment，但实际上真正展示的fragment是我们自己创建的“多余”的fragment。
- 在activity创建的时候根据fragment的TAG，查询有没有fragment已经存在了。  
- 尽量避免在activity的生命周期中去创建fragment。


感觉以上几个办法都各有问题，不是很完美，水平有限，如果谁有更好的办法欢迎和我讨论。

