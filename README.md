The systemd documentation is not quite as detailed as I would like it to be
regarding the interaction of some of the dependency directives and how the
parallel nature of unit startup, execution, and termination are affected by
these directives. This repository explores these details empirically through
experiments.

## `test-interrupt` directory

This directory explores the interaction of an instance service,
`test-instance@.service`, that is interruped by another, paired instance
service, `test-interrupt@.service`. The interruption is performed by manually
executing `systemctl stop` in a third service, `test-interrupt.service`.
`test-interrupt.service` is is responsible for stopping instances of
`test-instance@.service`, starting instances of `test-interrupt@.service`, and
then restarting instances of `test-instance@.service`.

The instance and interrupt services are grouped into targets through use of
`WantedBy` directives in the instance/interrupt services on target services.
Additionally, both the instance and interrupt services contain `PartOf`
directives on their associated target service, which should result in these
instances being stopped or restarted when the target service they are part of
is stopped or restarted.

### Results

Performed under

    $ systemctl --version
    systemd 219
    +PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 -SECCOMP +BLKID +ELFUTILS +KMOD +IDN

Log statements were produced with the following command

    $ journalctl -u test-instance\* -u test-interrupt\* -u test-interrupt.target -u test-interrupt.target --output short-precise

After enabiling all necessary services (instance and interrupt services) in
order to create wanted by symbolic links and instantiate the template units,
executing `systemctl start test-instance.target` resulted in the following log
statements

    A Oct 16 20:06:55.703738 localhost systemd[1]: Started test-instance@test1.service.
    B Oct 16 20:06:55.703842 localhost systemd[1]: Starting test-instance@test1.service...
      Oct 16 20:06:55.703859 localhost systemd[1]: Reached target test-instance.target.
    C Oct 16 20:06:55.703892 localhost systemd[1]: Starting test-instance.target.

This is as would be expected given the `WantedBy` directive in
`test-instance@test1.service`, creating a dependency from
`test-instance.target`.

It's not clear why the started message (A) is emitted before the starting
message (B), and why there is a starting message for the target (C), but no
started message.

Later executing `systemctl start test-interrupt.service` resulted in the
following log statements

      Oct 16 20:11:26.972244 localhost systemd[1]: Starting test-interrupt.service...
      Oct 16 20:11:26.974917 localhost systemd[1]: Stopped target test-instance.target.
      Oct 16 20:11:26.974953 localhost systemd[1]: Stopping test-instance.target.
    A Oct 16 20:11:26.975076 localhost systemd[1]: Stopping test-instance@test1.service...
      Oct 16 20:11:26.981661 localhost systemd[1]: Starting test-interrupt@test1.service...
    B Oct 16 20:11:56.983707 localhost systemd[1]: Started test-interrupt@test1.service.
      Oct 16 20:11:56.984105 localhost systemd[1]: Reached target test-interrupt.target.
      Oct 16 20:11:56.984133 localhost systemd[1]: Starting test-interrupt.target.
      Oct 16 20:11:56.988166 localhost systemd[1]: Stopped target test-interrupt.target.
      Oct 16 20:11:56.988205 localhost systemd[1]: Stopping test-interrupt.target.
    C Oct 16 20:12:26.979606 localhost systemd[1]: Started test-instance@test1.service.
      Oct 16 20:12:26.979732 localhost systemd[1]: Starting test-instance@test1.service...
      Oct 16 20:12:26.979750 localhost systemd[1]: Reached target test-instance.target.
      Oct 16 20:12:26.979785 localhost systemd[1]: Starting test-instance.target.
    D Oct 16 20:12:26.982078 localhost systemd[1]: Started test-interrupt.service.

These messages indicate that

* `test-instance@test1.service` (A) is stopped after `test-instance.target`, as
  would be expected given the `PartOf = test-instance.target` directive in
  `test-instance@.service`.

* A `oneshot` service is not marked as started until after execution returns
  from the command(s) executed by the `ExecStart` directive(s):
  `test-interrupt@test1.service` executes `sleep` with a timeout of 30 seconds
  and is not marked as started until 30 seconds after its starting message (B);
  and `test-interrupt.service` is not marked as started until all of the
  commands in all of its `ExecStart` directives have returned (D).

* A service will not be restarted until it has finished stopping:
  `test-instance@test1.service` is not marked as started until after the 60
  second sleep timeout in its `ExecStopPost` has elapsed (C).

* Services stopped as the result of a `PartOf` directive do not block
  `systemctl stop` from returning, nor do `ExecStopPost` directives:
  `test-instance@test1.service` is stopped as a result of its `PartOf`
  dependency on `test-instance.target` (A), but this does not stop systemctl
  from executing the next `ExecStart` directive in `test-interrupt.service` (B
  and the message immediatly before B).
