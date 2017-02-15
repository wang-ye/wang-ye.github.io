Practical Rails Debugging Tips

A lot of the Ruby stuff are quite old. 

Using Pry
Start with the Pry cheetsheet: https://gist.github.com/lfender6445/9919357

  !!!                Alias for `exit-program`
  !!@                Alias for `exit-all`
  $                  Alias for `show-source`
  ?                  Alias for `show-doc`
  @                  Alias for `whereami`
  clipit             Alias for `gist --clip`
  file-mode          Alias for `shell-mode`
  history            Alias for `hist`
  jist               Alias for `gist`
  quit               Alias for `exit`
  quit-program       Alias for `exit-program`
  reload-method      Alias for `reload-code`
  show-method        Alias for `show-source`

play + wtf?
@ to show current code
$ to show source

Setting breakpoints is an art.

cat --ex

ActiveRecord and Transactions


ActiveRecord::Base.logger = Logger.new(STDOUT)

Rspecs & FactoryGirl