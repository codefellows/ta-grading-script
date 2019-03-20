# Ta Grading Script for Tampermonkey

## Installation

1. Install the Tampermonkey chrome extension via the [chrome webstore](https://chrome.google.com/webstore/detail/tampermonkey/dhdgffkkebhmkfjojejmpbldmpobfkfo?hl=en).
1. Open the Tampermonkey dashboard.
1. Click the tab with a "+" sign to install a new userscript
1. Delete the contents of the script
1. Paste in this code and save

```
// ==UserScript==
// @name         Code Fellows Grading Script
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  create a directory (if not exists) for the student's work and clone their repository that is to be graded into it, then install any npm packages needed
// @author       Nicholas Carignan
// @match        https://github.com/*/*/pull/*
// @grant        GM_setClipboard
// ==/UserScript==

( function () {
  'use strict';
  // Keep track of the output commands
  const outputCommands = [];

  // Add button to page
  const button = document.createElement( 'button' );
  button.textContent = 'Clone';
  button.className = 'btn btn-sm';
  document.body
    .getElementsByClassName( 'gh-header-title' )[ 0 ]
    .appendChild( button );

  // When the user clicks the button copy the commands
  button.onclick = function () {
    // Get the user, repo and pull number from the url
    const [ _, user, repo, _1, pullNumber ] = window.location.pathname.split( '/' );

    // Make a request to the github api
    fetch( `https://api.github.com/repos/${user}/${repo}/pulls/${pullNumber}` )
      .then( response => {
        return response.json().then( data => {
          if ( response.status !== 200 ) {
            throw new Error( data.message );
          }
          return data;
        } );
      } )
      .then( data => {
        // Make a grading directory in cas it does not exist
        outputCommands.push( `mkdir ~/cf` );
        outputCommands.push( `mkdir ~/cf/ta` );
        outputCommands.push( `mkdir ~/cf/ta/grading` );

        // Cd into the grading repo;
        outputCommands.push( `cd ~/cf/ta/grading/` );

        //make a directory for the student's code and cd into it;
        outputCommands.push(`mkdir ${data.base.repo.owner.login} && cd ${data.base.repo.owner.login}`);


        // Clear the most recent repo you were grading and clone the latest version
        outputCommands.push( `rm -rf ${repo}` );
        outputCommands.push( `git clone https://github.com/${data.base.repo.owner.login}/${data.base.repo.name}` );
        outputCommands.push( `cd ${repo}` );

        // If the pull request originated from a master branch, checks out a custom branch called not-master
        const localBranchName = data.head.ref === 'master' ?
          'not-master' :
          data.head.ref;

        // Create the command to fetch and checkout the specific pull request
        outputCommands.push( `git fetch origin pull/${data.number}/head:${localBranchName}` );
        outputCommands.push( `git checkout ${localBranchName}` );

        // Cd into the pull request branch name : makes grading code challenges quicker if student adheres to naming conventions of branches and folders
        outputCommands.push( `cd ${localBranchName}` );

        //Install node packages
        outputCommands.push( `npm i` );

        // Insert any Environment Variables you may need here in the form of ``` outputCommands.push( `export <env variable name>=<env variable value>` ); ```


        // Insert any automated testing you would like to run here


        // Join the output commands togather so they can be run
        GM_setClipboard( outputCommands.join( '; ' ) );
        alert( 'Commands copied to clipboard' );
      } )
      .catch( err => {
        alert( `Something went wrong:\n  ${err.message}` );
      } );
  };
} )();

```
