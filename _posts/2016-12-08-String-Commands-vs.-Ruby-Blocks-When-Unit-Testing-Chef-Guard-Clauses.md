---
layout: post
title: Command Strings vs. Ruby Blocks When Unit Testing Chef Guard Clauses
---

### Chef Guards

The great thing about the major Chef resources (`file`, `directory`, `package`, etc.) is that they're *idempotent*, meaning that they will produce the same effect if executed more than once. Chef takes care of this under the hood. If, however, you find yourself needing to use one of the resources that executes arbitrary scripts, like `execute` or `bash`, you need to implement idempotency yourself with the use of [guards](https://docs.chef.io/resource_common.html#guards). A guard describes the condition when your script should (`only_if`) or shouldn't (`not_if`) be executed.

When working with an `execute` resource, the guard methods will accept either a **command string**, a Ruby string interpreted as a shell command, or a **Ruby block** (surrounded by `{}` or `do...end`). Let's take a look at an example `execute` resource that uses a guard to specify that the script should be executed as long as a certain `lockfile` **does not exist**:

```ruby
# command string
execute 'say hello' do 
  command 'echo "hello world"'

  not_if '[[ -f /tmp/lockfile]]'
end

# Ruby block
execute 'say hello' do 
  command 'echo "hello world"'
  
  not_if { ::File.exist?('/tmp/lockfile') }
end
```

I am much more comfortable writing Ruby than bash, so I tend to prefer the second form for purposes of readability - and especially if the guard is more complicated than a simple check like the one above! 

### Unit Tests with ChefSpec: Guards

Unit tests are a powerful way to increase our confidence in the correctness of custom logic like the above guards, and it's here that we'll see that the Ruby block method isn't quite so simple when the tests are considered alongside the implementation. Let's start with a simple test of the command string version:

```ruby
# spec/unit/recipes/default_spec.rb

describe 'test::default' do
  let(:chef_run) do
    runner = ChefSpec::SoloRunner.new
    runner.converge(described_recipe)
  end

  context 'when the lockfile is absent' do
    it 'executes the script' do
      expect(chef_run).to run_execute('say hello')
    end
  end
end
```

When we run this, test we'll get the following failure:

```
$ chef exec rspec spec/unit/recipes/default_spec.rb
Failures:

  1) test::default when the lockfile is absent executes the script
     Failure/Error: runner.converge(described_recipe)

     ChefSpec::Error::CommandNotStubbed:
       Executing a real command is disabled. Unregistered command:

           command("[[ -f /tmp/lockfile ]]")

       You can stub this command with:

           stub_command("[[ -f /tmp/lockfile ]]").and_return(...)
```

Oops! ChefSpec doesn't let us run shell commands during tests and has shown us how to stub our guard. That's good because we would've wanted to use a stub here anyway. If you're unfamiliar with stubbing, it's a way to force part of your code to behave in a certain way during a test so that you can validate how the code behaves in particular circumstances. In this case, we would want to have one test that simulates the absence of the lockfile and one that simulates its presence. And, as ChefSpec has pointed out, stubbing a shell command is very simple: just use `stub_command`. Here's what the updated test looks like:

```ruby
context 'when the lockfile is absent' do
  it 'executes the script' do
    stub_command("[[ -f /tmp/lockfile ]]").and_return(false)

    expect(chef_run).to run_execute('say hello')
  end
end

context 'when the lockfile is present' do
  it 'executes the script' do
    stub_command("[[ -f /tmp/lockfile ]]").and_return(true)

    expect(chef_run).not_to run_execute('say hello')
  end
end
```

And with that, we're back in the green.

### Unit Tests with ChefSpec: Ruby Blocks

Now let's write tests for the Ruby block version of `execute` block. Rather than use `stub_command`, we'll need to rely on some good old-fashioned RSpec stubbing:

```ruby
# Just looking at a single test for the example's sake

context 'when the lockfile is absent' do
  it 'executes the script' do
    allow(File).to receive(:exists?).with('/tmp/lockfile').and_return(false)

    expect(chef_run).to run_execute('say hello')
  end
end
```

This time we get a different, and much more imposing, failure/error message:

```
  1) test::default when the lockfile is absent executes the script
     Failure/Error: runner.converge(described_recipe)

       #<File (class)> received :exists? with unexpected arguments
         expected: ("/tmp/lockfile")
              got: ("/var/folders/x2/k4j769z97zdbb7tcm871k3q80000gn/T/d20161202-48532-ggaf9q/cookbooks/test/.uploaded-cookbook-version.json")
        Please stub a default value first if message might be received with other args as well.
```

What's going on here? It turns out that we can't stube the `File` class in this way because `File` is also used by ChefSpec when it runs the tests and our stub is messing that up. So we have to make sure that `File` is only stubbed where it appears in the test:

```ruby
context 'when the lockfile is absent' do
  it 'executes the script' do
    allow(File).to receive(:exists?).and_call_original
    allow(File).to receive(:exists?).with('/tmp/lockfile').and_return(false)

    expect(chef_run).to run_execute('say hello')
  end
end

context 'when the lockfile is present' do
  it 'executes the script' do
    allow(File).to receive(:exists?).and_call_original
    allow(File).to receive(:exists?).with('/tmp/lockfile').and_return(true)

    expect(chef_run).not_to run_execute('say hello')
  end
end
```

The line ending with `.and_call_original` tells ChefSpec that if `File::exists?` is called *with any arguments other than the one supply on the next line*, that it should "call the original," i.e. act in that case as if the stub didn't exist.

### DRYing Up the `File` Stub

As you can see, we now have four lines of stubs to accomplish the same thing that took just two with the first example. We can DRY this up a bit by putting the `.and_call_original` line in a `before` block outside of both `context` blocks:

```ruby
before do
  allow(File).to receive(:exists?).and_call_original
end

context 'when the lockfile is absent' do
  it 'executes the script' do
    allow(File).to receive(:exists?).with('/tmp/lockfile').and_return(false)

    expect(chef_run).to run_execute('say hello')
  end
end

context 'when the lockfile is present' do
  it 'executes the script' do
    allow(File).to receive(:exists?).with('/tmp/lockfile').and_return(true)

    expect(chef_run).not_to run_execute('say hello')
  end
end
```

And that about does it. At the end of the day, whether I decide to use command strings or Ruby blocks in my guards depends on several factors, mostly the complexity of the logic and the difficulty of testing. In this case, either one works reasonably well, but I find the simplicty of the string command with `stub_command` very appealing. The most important thing to be aware of, though, is what's going on when you use a regular RSpec stub (`allow`) and get unexpected errors because more is being stubbed than you realized.
