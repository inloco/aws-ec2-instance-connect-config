# AWS EC2 Instance Connect Configuration

This package contains the EC2 instance configuration and scripts necessary to enable AWS EC2 Instance Connect.

## AuthorizedKeysCommand

The AuthorizedKeysCommand is split into three parts:

* eic_run_authorized_keys is the main entry point and wraps the rest in a 5 second timeout
* eic_curl_authorized_keys, which is the entry point for sshd on an ssh call
* eic_parse_authorized_keys, which is the main authorized keys command logic

This split is intentional - as parse takes all necessary pieces as command inputs is can be unit tested independently.  curl, however, obviously needs to curl EC2 Instance Metadata Service and so cannot be tested without mocking the actual service.

### eic_curl_authorized_keys

The curl script verifies we are actually running on an EC2 instance and cURLs relevant information from EC2 Instance Metadata Service and send it to parse.

Note that it must make several curl commands to proceed.  If it cannot do so it fast-fails to prevent blocking the ssh daemon.

The command also queries several OCSP staples from EC2 Instance Metadata Service.
These OCSP staples are provided from the AWS EC2 Instance Connect Service to avoid needing to query each CRL or OCSP authority at ssh calltime as that would have major performance implications.
The staples are passed to and used by parse_authorized_keys to check certificate validity without the need for extra external calls.

### eic_parse_authorized_keys

In addition to the fields required to complete all the below process, a key fingerprint may be provided.  If a key fingerprint is specified then only ssh keys found matching that fingerprint will be returned.

#### Certificate Validation

The core idea behind AWS EC2 Instance Connect is that all ssh keys vended have been trusted by AWS.  This can be verified by checking each key's signature (more on that later) and vetting the signing certificate used.

The signing certificate goes through a deep verification flow that checks:

* The CN matches the service
* The certificate trust chain can be verified up through the Amazon Trust Services Certificate Authority
* The entire certificate chain is valid - ie, none of them have been revoked

The first and second are done via standard openssl checks.  The third, however, does not query the relevant CRLs or OCSP authorities at runtime to avoid adding a network call to sshd.
Instead, OCSP staples are expected to be provided by the invoker (i.e., eic_curl_authorized_keys).
As OCSP staples are cryptographically signed and can be verified against a trusted authority these are considered a sufficient validity check.

#### Key Processing

The keys are expected to be presented to the script in the format

<pre>
# Key Metadata
# Key Metadata
# Key Metadata
[ssh key]
signature
</pre>

Currently, the expected metadata is, in order,

1. Expiration timestamp
2. Instance ID
3. IAM Caller
4. Request ID

Once this data has been loaded, the following checks are run:

* Has the expiration timestamp passed?
* Is the instance ID correct?
* Does the signature match the provided data?
* If a specific key fingerprint was provided to search for, does this key match that fingerprint?

The signature is specifically expected to be for the *entire* key blob - all metadata entries plus the ssh key.  It should have been generated by the AWS EC2 Instance Connect service's private key, which is verified using the vetted signing certificate.

If all of these checks pass then the key will be presented to the ssh daemon.  Otherwise, it will be ignored and the next key will be processed.

Any time a key is provided to the ssh daemon it will be logged to the system authpriv log for auditing purposes.

## Host Key Harvesting

The fourth script, eic_harvest_hostkeys, has no direct relation to the AuthorizedKeysCommand.  Instead, it is used to pass the instance's ssh host keys back to the EC2 Instance Connect Service.
This is necessary to establish trust for the EC2 Console for the console's one-click in-browser ssh terminal feature.

### systemd Module

The systemd module provided for host key harvesting is a basic one-shot to invoke eic_harvest_hostkeys.

### eic_harvest_host_keys

The host key harvesting script has to run similar logic to eic_curl_authorized_keys to get some basic information from Instance Metadata Service about the instance itself.
It then reads the ssh host keys on the machine, then creates and signs an AWS Signature Version 4 Request to 

## Unit Testing

As parse_authorized_keys requires a valid certificate, CA, and OCSP staples, unit testing is a somewhat involved process.

Fortunately, all of this has been automated to a convenient entry point: `bin/test/run_authorized_keys_tests.sh`.  This will

1. Invoke `bin/test/setup_certificates.sh` to generate a test CA and trust chain
2. Invoke `bin/test/generate_ocsp.sh` to generate OCSP staples for the certificates
3. Iterate over test/input/direct and test/input/unsigned and test all entries via `bin/test/test_authorized_keys.sh`, expecting the matching contents of test/expected-output

test/input/direct's contents are passed directly as-is as they are not expected to contain valid signatures.

test/input/unsigned's contents, however, are expected to get far enough in the process to (potentially) need a valid signature.  As such, signatures are generated on-the-fly using the pre-generated certificate's private key.

The structure of test/input/unsigned, rather than files, is directories to test.  The directory's contents should be in a numeric order that the test script will iterate.
Each file should have *one* test key blob to sign.
The actual test input will be the result of generating signatures for each file and constructing an imitation of the service's key bundle.

## End-to-End Testing

It is strongly recommended you build a sample package for end-to-end testing.  If you do you may skip steps 1-4 in favor of installing your test build.

1. Copy the contents of src/bin to a directory on an EC2 instance (such as /opt/aws/bin, see debian/postinst or rpmsrc/SPECS/generic.spec)
2. Create a system user for sshd to use as the AuthorizedKeysCommandUser, such as ec2-instance-connect (see debian/postinst or rpmsrc/SPECS/generic.spec for configuration)
3. Modify /etc/ssh/sshd_config so that you have
   * `AuthorizedKeysCommand /path/from/step/1/eic_run_authorized_keys`
   * `AuthorizedKeysCommandUser user-from-step-2`
4. Restart sshd (ie, sudo systemctl restart sshd (or ssh on Ubuntu))
5. Attempt to connect to your instance with EC2 Instance Connect

## Building RPM/Debian Packaging

If desired, this scripting can be added to an EC2 instance for further testing.  A convenience pair of scripts - bin/make_rpm.sh and bin/make_deb.sh - have been provided to quickly build test packages.
Each may be invoked via `make rpm` and `make deb` respectively.  Be sure to update VERSION!

**Debian packaging tools are intended for testing purposes only.  Please follow standard Debian process to submit patches.**

### Why is it generic.rpm?

"generic.rpm" is used internally to differentiate the specfile for Fedora/RHEL/CentOS from Amazon Linux.
There is a separate Amazon-proprietary rpmspec and build process optimized for Amazon Linux.

## Why UNIX shell?

This package is intended as a simple reference implementation of the instance-side pieces of the EC2 Instance Connect feature, though as time has gone on it has become much more complex than originally intended.
Amazon is considering reimplementation in another language for the future but for the time being we will continue to iterate on the shell implementation.

## License

This library is licensed under the Apache 2.0 License.
