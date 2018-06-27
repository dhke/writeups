This is based on [Torsten Zuehlsdorff's excellent writeup](https://github.com/t-zuehlsdorff/gitlabhq/blob/master/doc/install/installation-freebsd.md) for installing Gitlab CE from ports.

## Host setup for the jail

Set the required sysctls:

    # /etc/sysctl.conf
    security.jail.sysvipc_allowed=1


## Preparing the host's network

We put the jail into a private network behind a local host NAT

	# create lo1 as an interface and
	# add a local private IP addresses to it.
	# if you need more IP addresses for the jail,
	# just add more aliases.
	sysrc cloned_interfaces="lo1"
	sysrc ifconfig_lo1_alias="inet 10.0.0.1 netmask 255.255.255.240"

	# start the interface
	service netif start lo1

Setup the NAT using pf. Use whichever one of FreeBSD's firewall suits you best

    # /etc/pf.conf
    EXT_IF="igb0"
    JAIL_IF="lo1"
    NET_JAIL="10.0.0.0/28"

    # perform NAT for the jail
    nat pass on $EXT_IF \
		from $NET_JAIL \
		to any -> ($EXT_IF)


To enable incoming connections to the jail from outside, perform port redirection
on the external interface to the jail's IP. Enable this *after* you configured and
secured your gitlab installation.

	# rdr on $EXT_IF inet proto tcp \
	# 	to port ssh -> 10.0.0.1 port ssh
	# rdr on $EXT_IF inet proto tcp \
	# 	to port http -> 10.0.0.1 port http
	# rdr on $EXT_IF inet proto tcp \
	# 	to port https -> 10.0.0.1 port https

	pass quick on $JAIL_IF inet \
		no state
 
	pass quick on $JAIL_IF inet6 \
		no state


## Preparing the iocage jail 

    JAIL_RELEASE="11.1-RELEASE"

	# fetch the release for iocage
	# skip this, if you already have a current version
	# of the desired release installed
	iocage fetch -r "${JAIL_RELEASE}"

	iocage create -r "${JAIL_RELEASE}" -n gitlab ip4_addr=10.0.0.1 allow_sysvipc=1

## Setup PostgreSQL

    iocage console gitlab
	pkg install postgresql95-server 
	service postgresql initdb
	service postgresql start

	# temporarily make git superuser on the database.
	# make sure to drop this privileges later.
	createuser -s git
	createdb -O git gitlabhq_production
	
	# gitlab needs [pg_trgm](https://www.postgresql.org/docs/9.5/static/pgtrgm.html), enable it.
	psql -U git -d gitlabhq_production -c 'CREATE EXTENSION pg_trgm';

As for the remainer, follow Torsten Zuehlsdorff's writeup:

https://github.com/t-zuehlsdorff/gitlabhq/blob/master/doc/install/installation-freebsd.md
