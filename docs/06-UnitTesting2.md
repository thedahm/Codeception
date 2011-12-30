# Unit Testing 2

We already compared approach of writing tests provided by PHPUnit (and similar unit test libraries) and Codeception.
In this chapter we will lift up the curtains and show you a bit of magic Codeception does to simplify unit testing.

## What do we test

As we discovered that's quite an easy to define the class and method you are going to test. We take the $class parameter of Cest class, and method's name as a target method.

``` php
<?php
class PostCest {
	$class = 'Post';

	function save(CodeGuy $I) {
		// will test Post::save()
		// or Post->save()
	}

}
?>
```

Note, the CodeGuy object is passed as parameter into each test.

To redefine target of test consider using testMethod action. Note, you can't change the method you test inside of test. That's just simply wrong. So the testMethod call should be put in the very begining or ignored. 

``` php
<?php
class PostCase {
	$class = 'Post';

	function saveWithParameters(CodeGuy $I)
	{
		$I->testMethod('Post::save');
	}
}
?>

```
Also we recommend you to improve your test adding a short test definition with wantTo command, as we did it for acceptance tests.

``` php
<?php
class PostCase {
	$class = 'Post';

	function saveWithParameters(CodeGuy $I)
	{
		$I->wantTo('save post with different parameters');
		$I->testMethod('Post::save');
	}
}
?>
```

Now we need to understand basic principles of Codeception unit testing.

## Describe And Run

The thing you should understand very clearly: you write scenario, not the code which actually would be run. Usage of PHP-specific operators can lead to unpredictable results. Methods of Guy class just record the actions. They will be performed after test is fully written, and environment prepared. This leads us to some limitations we should keep in mind.

#### Code is executed before the test.

All the code you write without Guy class will be executed before the test is run. No matter where in test your code is written.

``` php
<?php

$I->testMethod('Comment::save');
$I->executeTestedMethod(array('post_id' => 5);
DB::updateCounters();
$I->seeInDatabase('posts',array('id' => 5, 'comments_count' => 1));

?>
```

This scenario will fail, because developer expects the comment counter will be incrementer by DB::updateCounters call. But this method will be executed before comment is saved, so the last assertions will fail. To perform actions inside the scenario add your code as an action into CodeHelper module. 

#### Actions won't return any results

No method of Guy class is allowed to return values. It will return the current CodeGuy instance only. Reconsider your testing scenario every time you want write something like this:

``` php
<?php
$I->testMethod('Post::insert');
$I->executeTestedMethod(array('title' => 'Top 10 kitties');
$post_id = $I->takeLastResult(); // this won't work
$I->seeInDatabase('posts', array('id' => $post_id));
?>
```

For testing result we use ->seeResult* actions. But you can't use data returned by tested method inside your next actions.

#### All variables must be constant.

ALl variables used in scenarios can't be changed. 

``` php
<?php
$post_id = 1;
$I->seeInDatabase('posts', $post_id);
$post_id = 3;
$I->seeInDatabase('posts', $post_id);
?>
```	

As we already know, the first assertions won't be executed as expected. The test will start with the $post_id equals to 3. This example could be rewritten with two post_id variables instead of one. 

#### Objects can be updated

Still, not all varibles are constant. CodeGuy can update objects inside scenarios.

``` php
<?php
$post = new Post(array('title' => 'Top 10 kitties'));
$I->testMethod('Post::save');

$I->expect('post about kitties created')
	->executeTestedMethodOn($post);
	->seeInDatabase('posts', array('title' => 'Top 10 kitties'));

$I->changeProperty($post, 'title', 'Top 10 doggies');

$I->expect('the kitties post is updated')
	->executeTestedMethodOn($post);
	->seeInDatabase('posts', array('title' => 'Top 10 doggies'))
	->dontSeeInDatabase('posts', array('title' => 'Top 10 kitties'));
?>
```

If we wrote test as a code, we would update the post with setters.
In Codeception we work directly with object's properties. If you think of it, you should conclude that approach makes sense. The setter's behavior may change in future, thus the test will fail, even the tested method itself is valid. 

### Making Mocks Dynamically

That's plenty much limitations that Codeception provides. But what's the profit of BDD approach into writing unit tests?

Let's go back to controller test example.

``` php
<?php
        $I->executeTestedMethodOn($controller, 1)
            ->seeResultEquals(true)
            ->seeMethodInvoked($controller, 'render');
?>

```

We test the invokation of 'render' method without any mock definition we did for PHPUnit. We just say: 'I see method invoked'. That's not of a testser's business how this method is mocked. Also, as we saw mock for this method can be changed on next call.

But how we can define mock and perform assertion the same time? We don't. Action 'seeMethodInvoked' only performs assertion the mocked method was run. The mock was created by 'executeTestedMethodOn' command. It looked through scenario and created mocks for methods execution of which we want to test.

You should no more think about creating mocks! 

Still, you have to use stub classes, in order to make dynamical mocking work.

``` php
<?php
$I->haveFakeClass($controller = Stub::makeEmpty('Controller'));
// same as
$I->haveStub($controller = Stub::makeEmpty('Controller'));
?>
```

Only the objects defined by one of those methods can be turned into mocks. 
For stubs that won't become mocks, haveFakeClass execution is not required. 

### setUp and tearDown

Cest files has analogs for PHPUnit's setUp and tearDown methods. 
You can use _before and _after methods of Cest class to prepare and clean environment.

``` php
<?php
class ControllerCest {
	$class = 'Controller';

	public function _before() {
		$this->db = Stub::makeEmpty('DbConnector');
	}

	public function show(CodeGuy $I)
	{
		$controller = Stub::makeEmptyExcept('Controller', 'save');
		$I->setProperty($controller, 'db', $this->db);
		// ...
	}

	public function _after() {		
	}
}
?>
```

## Working with Database

We used Db module widely in examples. As we tested methods of model, it was natural to test result inside the database.
Connect the Db module to your unit suite to perform 'seeInDatabase' calls. 
But before each test the database should be cleaned. Deleting all tables and loading dump may take quite a lot of time. For unit tests that are supposed to be run fast that's a catastrophee. 

We could perform all our database interactions inside the transaction, but MySQL doesn't support nested transaction. We recommend you to using SQLite Memory instead MySQL database for running your unit tests.

ORMs like Doctrine or Doctrine2 can emulate nested transaction. Thus, If you use such ORMs, you'd better connect their modules to your suite.  In order to not conflict with Db module they have slightly different actions for looking into database.

``` php
<?php
// For Doctrine2
$I->seeInRepository('Entity',array('property' => 'value'));
// For Doctrine1
$I->seeInTable('Table',array('property' => 'value'));
?>
```

## Conclusion

Codeception has it's powers and it's limits. We belive Codeception limits keeps your tests clean and narrative. Codeception hardens writing a bad code for tests. Codeception has it's simple but powerful tools to create stubs and mocks. Different modules can be attached to unit tests, which, for example, will simplify database interactions. 